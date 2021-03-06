<template>
  <div v-if="showPipelinesOnly ? (pipelineCount > 0) : true" :class="['project-card', status]">
    <div class="content">
      <div class="title small">{{ project !== null ? project.namespace.name : '...' }} /</div>
      <a class="title" target="_blank" rel="noopener noreferrer" :href="project !== null ? project.web_url : '#'">
        {{ project !== null ? project.name : 'Loading project...' }}
      </a>
      <div class="pipeline-container">
        <em v-if="pipelines !== null && pipelineCount === 0" class="no-pipelines">
          No recent pipelines
        </em>
        <div v-else-if="pipelines !== null">
          <template v-for="refName in refNames">
            <div v-for="(pipeline, index) in pipelines[refName]" :key="pipeline.id">
              <pipeline-view :pipeline="pipeline" :project="project" :show-branch="index === 0" />
            </div>
          </template>
        </div>
        <octicon v-else name="sync" scale="1.4" spin />
      </div>
    </div>
    <div class="spacer"></div>
    <div class="info">
      <div class="spacer"></div>
      <gitlab-icon class="calendar-icon" name="calendar" size="12" />
      <timeago v-if="project !== null" :since="project.last_activity_at" :auto-update="1"></timeago>
      <time v-else>...</time>
    </div>
  </div>
</template>

<script>
  import Octicon      from 'vue-octicon/components/Octicon';
  import 'vue-octicon/icons/clock';
  import 'vue-octicon/icons/git-branch';
  import 'vue-octicon/icons/sync';
  import Config       from '../Config';
  import GitlabIcon   from './gitlab-icon';
  import PipelineView from './pipeline-view';

  export default {
    components: {
      GitlabIcon,
      PipelineView,
      Octicon
    },
    name: 'project-card',
    props: ['project-id'],
    data: () => ({
      project: null,
      pipelines: null,
      pipelineCount: 0,
      refNames: [],
      status: '',
      loading: false,
      refreshInterval: null
    }),
    computed: {
      showMerged() {
        let configuredShowMerged = Config.root.projectFilter['*'].showMerged;
        if (Config.root.projectFilter.hasOwnProperty(this.project.path_with_namespace)) {
          configuredShowMerged = Config.root.projectFilter[this.project.path_with_namespace].showMerged;
        }
        return configuredShowMerged;
      },
      showTags() {
        let configuredShowTags = Config.root.projectFilter['*'].showTags;
        if (Config.root.projectFilter.hasOwnProperty(this.project.path_with_namespace)) {
          configuredShowTags = Config.root.projectFilter[this.project.path_with_namespace].showTags;
        }
        return configuredShowTags;
      },
      showPipelinesOnly() {
        return Config.root.pipelinesOnly;
      }
    },
    mounted() {
      this.fetchProject();
    },
    beforeDestroy() {
      if (this.refreshIntervalId) clearInterval(this.refreshIntervalId);
    },
    watch: {
      project() {
        this.fetchPipelines();
      },
      pipelines: {
        deep: true,
        handler(pipelines) {
          if (!this.project) {
            this.status = '';
            this.refreshInterval = 60000;
            return;
          }

          let configuredDefaultBranch = Config.root.projectFilter['*'].default || this.project.default_branch;

          if (Config.root.projectFilter.hasOwnProperty(this.project.path_with_namespace)) {
            configuredDefaultBranch = Config.root.projectFilter[this.project.path_with_namespace].default || this.project.default_branch;
          }

          if (
            pipelines &&
            this.project &&
            !!pipelines[configuredDefaultBranch] &&
            pipelines[configuredDefaultBranch].length > 0
          ) {
            this.status = pipelines[configuredDefaultBranch][0].status;

            switch (pipelines[configuredDefaultBranch][0].status) {
              case 'pending':
              case 'running':
                this.refreshInterval = 5000;
                break;
              default:
                this.refreshInterval = 15000;
            }
          } else {
            this.status = '';
            this.refreshInterval = 60000;
          }
        }
      },
      refreshInterval(newInterval, oldInterval) {
        if (newInterval !== oldInterval) {
          if (this.refreshIntervalId) clearInterval(this.refreshIntervalId);
          this.refreshIntervalId = setInterval(() => {
            if (!this.loading) {
              this.fetchProject();
            }
          }, newInterval);
        }
      }
    },
    methods: {
      async fetchProject() {
        this.loading = true;

        this.project = await this.$api(`/projects/${this.projectId}`);
        this.$emit('input', this.project.last_activity_at);

        this.loading = false;
      },
      async fetchPipelines() {
        this.loading = true;

        const maxAge = Config.root.maxAge;
        const showMerged = this.showMerged;
        const showTags = this.showTags;
        const fetchCount = Config.root.fetchCount;

        const branches = await this.$api(`/projects/${this.projectId}/repository/branches`, {
            per_page: fetchCount > 100 ? 100 : fetchCount
          }, {follow_next_page_links: fetchCount > 100});
        const branchNames = branches.filter(branch => showMerged ? true : !branch.merged)
                                    .sort((a, b) => new Date(b.commit.committed_date).getTime() - new Date(a.commit.committed_date).getTime()).reverse()
                                    .map(branch => branch.name)
                                    .filter(branchName => {
          let filter = Config.root.projectFilter['*'];

          if (Config.root.projectFilter.hasOwnProperty(this.project.path_with_namespace)) {
            filter = Config.root.projectFilter[this.project.path_with_namespace];
          }

          return !!branchName.match(new RegExp(filter.include)) &&
            (!filter.exclude || !branchName.match(new RegExp(filter.exclude)));
        });
        let tags = [];
        if (showTags) {
          tags = await this.$api(`/projects/${this.projectId}/repository/tags`, {
              per_page: fetchCount > 100 ? 100 : fetchCount
            }, {follow_next_page_links: fetchCount > 100});
        }
        const tagNames = tags.map((tag) => tag.name)
        const newPipelines = {};
        let count = 0;
        const refNames = branchNames.concat(tagNames)
        for (const refName of refNames) {
          const pipelines = await this.$api(`/projects/${this.projectId}/pipelines`, {
            ref: refName,
            per_page: fetchCount > 100 ? 100 : fetchCount
          }, {follow_next_page_links: fetchCount > 100});

          const resolvedPipelines = [];

          if (pipelines.length > 0) {
            const filteredPipelines = [];

            for (const pipeline of pipelines) {
              if (pipeline.status === 'pending' || pipeline.status === 'running') {
                filteredPipelines.push(pipeline);
              }
            }

            for (const pipeline of filteredPipelines) {
              const resolvedPipeline = await this.$api(`/projects/${this.projectId}/pipelines/${pipeline.id}`);
              if ((maxAge === 0 || ((new Date() - new Date(resolvedPipeline.updated_at)) / 1000 / 60 / 60 <= maxAge))) {
                resolvedPipelines.push(resolvedPipeline);
              }
            }

            if (pipelines.length >= 1 && filteredPipelines.length === 0) {
              const resolvedPipeline = await this.$api(`/projects/${this.projectId}/pipelines/${pipelines[0].id}`);
              if ((maxAge === 0 || ((new Date() - new Date(resolvedPipeline.updated_at)) / 1000 / 60 / 60 <= maxAge))) {
                newPipelines[refName] = [resolvedPipeline];
                count++;
              }
            } else {
              newPipelines[refName] = resolvedPipelines;
              count += resolvedPipelines.length;
            }
          }
        }

        this.pipelines = newPipelines;
        this.refNames = refNames;
        this.pipelineCount = count;
        this.loading = false;
      }
    }
  };
</script>

<style lang="scss" scoped>
  .project-card {
    margin: 4px;
    border-radius: 3px;
    background-color: #424242;
    flex-grow: 1;
    display: flex;
    flex-direction: column;
    transition: background-color 200ms;

    &.success {
      background-color: #2E7D32;
    }

    &.running {
      background-color: #1565C0;
    }

    &.pending {
      background-color: #A93F00;
    }

    &.failed {
      background-color: #C62828;
    }

    &.canceled {
      background-color: #010101;
    }

    &.skipped {
      background-color: #4b4b4b;
    }

    .content {
      padding: 12px;

      .title {
        white-space: nowrap;
        font-size: 16px;
        font-weight: bold;
        text-shadow: 1.5px 1.5px rgba(0, 0, 0, 0.4);
        text-decoration: none;
        color: inherit;

        &.small {
          font-size: 12px;
          line-height: 0.6;
        }
      }

      .pipeline-container {
        padding: 8px 0 0 0;
      }

      .no-pipelines {
        color: rgba(255, 255, 255, 0.5);
        font-size: 10px;
      }
    }

    .spacer {
      flex-grow: 1;
    }

    .info {
      padding: 12px;
      border-top: 1px solid rgba(255, 255, 255, 0.1);
      display: flex;
      align-items: center;
      font-size: 12px;
      color: rgba(255, 255, 255, 0.3);

      time {
        line-height: 1;
      }

      .calendar-icon {
        margin-right: 4px;
      }
    }
  }
</style>
