<template>
  <div id="projects">
    <nav class="navbar navbar-expand-lg navbar-light bg-dark">
      <router-link
        class="navbar-brand text-light"
        to="/"
      >MorphoCluster</router-link>
      <ul class="navbar-nav mr-auto">
        <li class="nav-item active text-light">
          Projects
        </li>
      </ul>
    </nav>
    <div class="scrollable">
      <div class="container">
        <div
          class="alerts"
          v-if="alerts.length"
        >
          <b-alert
            :key="a"
            v-for="a of alerts"
            dismissible
            show
            :variant="a.variant"
          >
            {{a.message}}
          </b-alert>
        </div>
        <b-table
          id="projects_table"
          striped
          sort-by="name"
          :items="projects"
          :fields="fields"
          showEmpty
        >
          <template slot="table-colgroup">
            <col class="col-wide" />
            <col class="col-narrow" />
          </template>
          <template v-slot:cell(name)="data">
            <router-link :to="{name: 'project', params: {project_id: data.item.project_id}}">{{data.item.name}}</router-link>
          </template>
           <template v-slot:cell(progress)="data">
            <b-progress
              v-if="'progress' in data.item"
              :max="data.item.progress.leaves_n_nodes"
            >
              <b-progress-bar
                variant="success"
                :value="data.item.progress.leaves_n_filled_nodes"
                v-b-tooltip.hover
                :title="`${data.item.progress.leaves_n_filled_nodes} / ${data.item.progress.leaves_n_nodes} filled up`"
              />
              <b-progress-bar
                variant="warning"
                :value="data.item.progress.leaves_n_approved_nodes - data.item.progress.leaves_n_filled_nodes"
                v-b-tooltip.hover
                :title="`${data.item.progress.leaves_n_approved_nodes} / ${data.item.progress.leaves_n_nodes} approved`"
              />
            </b-progress>
          </template>
          <template v-slot:cell(action)="data">
            <b-button
              size="sm"
              variant="primary"
              class="mr-2"
              :to="{name: 'approve', params: {project_id: data.item.project_id}}"
            >
              Validate
            </b-button>
            <b-button
              size="sm"
              variant="primary"
              class="mr-2"
              :to="{name: 'bisect', params: {project_id: data.item.project_id}}"
            >Grow</b-button>
            <b-button
              size="sm"
              variant="primary"
              class="mr-2"
              @click.prevent="showSaveModal(data.item)"
            >Save</b-button>
          </template>
          <template v-slot:empty>
            <div class="text-center">No projects available.</div>
          </template>
        </b-table>
        <p v-if="!projects">No projects available.</p>
        <div style="margin: 0 auto; width: 0;">
          <b-button
            size="sm"
            variant="danger"
            class="mr-2"
            href="/labeling"
          >
            Expert mode
          </b-button>
        </div>
      </div>
    </div>
    <b-modal
      ref="saveModal"
      lazy
      centered
      :title="save_title"
      @ok="HandleSaveOk"
    >
      <div class="d-block text-center">
        <form @submit.stop.prevent="HandleSaveOk">
          <b-container fluid>
            <b-form-row class="my-1">
              <b-col sm="3">
                <label for="form_save_name">Name:</label>
              </b-col>
              <b-col sm="9">
                <b-form-input
                  type="text"
                  placeholder="Enter a name for the saved file"
                  id="form_save_name"
                  v-model="save_slug"
                ></b-form-input>
              </b-col>
            </b-form-row>
            <b-row
              class="my-3"
              v-if="save_saving"
            >
              <b-col sm="12">
                Saving...
              </b-col>
            </b-row>
          </b-container>
        </form>
      </div>
    </b-modal>
  </div>
</template>

<script>
import * as api from "@/helpers/api.js";

export default {
    name: "projects",
    props: {},
    components: {},
    data() {
        return {
            fields: [
                //{ key: "project_id", sortable: true },
                { key: "name", sortable: true },
                "progress",
                "action"
            ],
            projects: [],
            save_slug: "",
            save_title: "",
            save_project_id: null,
            save_saving: false,
            alerts: [],
        };
    },
    methods: {
        showSaveModal(project) {
            console.log("project", project);
            this.save_slug = project.name;
            this.save_project_id = project.project_id;
            this.save_title = `Save ${project.name} (${this.save_project_id})`;

            this.$refs.saveModal.show();
        },
        HandleSaveOk(evt) {
            evt.preventDefault();

            console.log("Saving", this.save_project_id, "...");
            this.save_saving = true;
            api.saveProject(this.save_project_id).then(result => {
                console.log("Project saved: " + result["tree_fn"]);
                this.save_saving = false;
                this.$nextTick(() => {
                    // Wrapped in $nextTick to ensure DOM is rendered before closing
                    this.$refs.saveModal.hide();
                });
            });
        }
    },
    mounted() {
        // Load node info
        api.getProjects()
            .then(projects => {
                this.projects = projects;

                return Promise.all(
                    this.projects.map(p => {
                        return api.getNodeProgress(p.node_id).then(progress => {
                            console.log(`Got progress for ${p.node_id}.`);
                            this.$set(p, "progress", progress);
                        });
                    })
                );
            })
            .catch(e => {
                console.log(e);
                this.alerts.unshift({
                    message: e.message,
                    variant: "danger"
                });
            });
    }
};
</script>

<style>
#projects {
    display: flex;
    flex-direction: column;
    align-items: stretch;
    flex: 1;
    overflow: hidden;
}

#projects_table tr td:nth-child(1) {
    width: 100%;
}

#projects_table tr td:not(:nth-child(1)) {
    width: auto;
    text-align: right;
    white-space: nowrap;
}

.scrollable {
    overflow-y: auto;
}

.alerts {
    padding-top: 1em;
}
</style>
