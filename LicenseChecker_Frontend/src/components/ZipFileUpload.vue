<template>
  <q-page class="custom-content">

    <!-- Section for uploading zip files -->
    <div v-if="selectedOption === 'ZipFileUpload'">
      <div v-show="!showTable" class="center-container custom-background-color">
        <div class="form-container">
          <q-form @submit="uploadZipFile()">
            <q-uploader hide-upload-btn label="Upload zip file" accept=".zip" @added="handleFileUpload"
              color="secondary">
            </q-uploader>

            <q-btn style=" margin-top: 15px; background-color:#1A8917; text-transform:capitalize; color: white;"
              :label="loading ? 'Uploading...' : 'Submit'" type="submit" :loading="loading" :disable="loading || !file" />
          </q-form>
          <q-banner v-if="fileError" dense inline-actions class="text-secondary bg-red q-mt-lg ">
            {{ fileError }}
            <template v-slot:action>
              <q-btn flat color="secondary" label="Dismiss" @click="dismissError" />
            </template>
          </q-banner>
        </div>

      </div>
      <!-- Table showing found licenses from uploaded zip file -->
      <div class="q-pa-md" v-show="showTable">
        <q-table :rows="dependencyLicenses" :columns="columns" row-key="package_name">
          <template v-slot:body="props">
            <q-tr :props="props">
              <q-td v-for="col in props.cols" :key="col.name" :props="props">
                <span v-if="col.name === 'selected'">
                  <q-checkbox v-model="props.row.selected" />
                </span>
                <span v-else-if="col.name === 'package_name'">
                  {{ props.row.package_name }}
                </span>
                <span v-else-if="col.name === 'license_name'">
                  {{ props.row.license_name }}
                </span>
                <span v-else-if="col.name === 'dropdown'">
                  <template v-if="Array.isArray(props.row.license_id) && props.row.license_id.length > 1">
                    <q-select v-model="props.row.dropdown" :options="props.row.license_id" outlined dense
                      style="width: 145px;" />
                  </template>
                  <template v-else>
                    {{ props.row.license_id[0] }}
                  </template>
                </span>
                <span v-else>
                  {{ col.value }}
                </span>
              </q-td>
            </q-tr>
          </template>
        </q-table>
        <div class="q-mt-md">
          <q-btn v-if="!fromLR" label="Find Compatible Licenses" @click="saveSelected" class="btn q-mr-xs" />
          <q-btn v-if="fromLR" @click="updateSelected" label="Add to Compatiblity list" class="btn q-mr-xs" />
          <q-btn label="Back" @click="goBack" color="secondary" />
        </div>
      </div>

    </div>

  </q-page>
</template>

<script>

import axios from 'axios';
import { mapGetters, mapActions } from 'vuex'

export default {
  name: "ZipFileUpload",
  data() {
    return {
      showTable: false,
      loading: false,
      file: null,
      dependencyLicenses: [],
      selectedLicenseIds: [],
      selectedPermissiveLicenses: {},
      selectedCopyleftLicenses: {},
      selectedLicenses: {},
      fileError: null,
      selectedOption: 'ZipFileUpload',
      options: [
        { value: 'ZipFileUpload', label: 'Zip File Upload' },
      ],
      columns: [
        { name: 'selected', label: 'Select', align: 'left' },
        { name: 'package_name', label: 'Package Name', align: 'left' },
        { name: 'license_name', label: 'License Name', align: 'left' },
        { name: 'dropdown', label: 'License ID', align: 'left' },
      ],
      licenseColumns: [
        { name: 'checkbox', label: 'Select', align: 'left' },
        { name: 'license', label: 'License Name', align: 'left' }
      ],
      fromLR: false,
      status: " ",
      name: " ",
      toggle: 0,
      checkTimer: null,
      repoName: " ",
      softwareid: null,
      uploadSuccess: false,
      licenses: null,
      checkboxSelection: {},
      responseArray: [],
      showDiv1: true,
      scrollPosition: 0,
      savedState: null,
    };
  },

  computed: { // accessing the state from store
    ...mapGetters(['getSelectedOption', 'getShowDiv1', 'getLicenses']),



  },

  beforeRouteEnter(to, from, next) {
    next((vm) => {
      vm.fromLR = from.name === 'LicenseRecommendation';
    });
  },
  methods: {
    handleFileUpload(files) {
      const maxSize = 50 * 1024 * 1024; // 50MB
      if (files[0].size > maxSize) {
        this.$q.notify({
          message: 'File too large',
          caption: 'The selected file exceeds the 50MB limit. Please upload a smaller file.',
          type: 'negative',
          position: 'center',
          timeout: 5000,
        });
        this.file = null;
        return;
      }
      this.file = files[0];
    },
    dismissError() {
      this.fileError = '';
    },
    getTooltipContent(value) {
      switch (value) {
        case 'github':
          return 'Retrieve code from github';
        case 'DependencyFileUpload':
          return 'Upload dependencies (upload a dependency file like a requirements.txt or a project.toml)';
        case 'ZipFileUpload':
          return 'Upload a zip file containing your project to analyze its licenses';
        case 'AddLicensesManually':
          return 'Select licenses from the list of Premissive and Copyleft licenses';
        default:
          return '';
      }
    },
    logValues() {
      console.log('selectedOption:afer ', this.selectedOption);
      console.log('showDiv1:', this.showDiv1);
      console.log('licenses:', this.licenses);
    },
    ...mapActions(['updateSelectedOption', 'updateShowDiv1', 'updateLicenses']),
    goToAddLicenses(actionType) {
      // Store the current state before navigating away
      this.updateSelectedOption(this.selectedOption);
      this.updateShowDiv1(this.showDiv1);
      this.updateLicenses(this.licenses);

      console.log("Selected Option", this.selectedOption);
      console.log("Showdiv1", this.showDiv1);
      console.log("Available licenses", this.licenses);



      // Navigate to DependencyFileUpload.vue
      if (actionType === 'fromDependencyFile') {
        this.$router.push('/DependencyFileUpload');
      } else if (actionType === 'manually') {
        this.$router.push('/AddLicensesManually');
      }

    },

    generateSoftwareid() {
      this.softwareid = this.file?.name;
      console.log("Software ID", this.softwareid);
      return this.softwareid;
    },
    getStatus() {
      console.log("In status", this.generateSoftwareid())

      axios
        .get(
          this.engineURL + "/api/v1/software/status/" +
          this.generateSoftwareid()
        )

        .then((response) => {
          this.status = response.data;
          console.log("Status", response.data);
          if (response.data.status) {
            // checking if it already has a status, i.e "Finished"
            this.status = response.data.status;
            this.$q.loading.show({
              message: this.status,
              boxClass: "bg-grey-2 text-secondary",
              spinnerColor: "secondary",
            });
            if (this.checkTimer) {
              setTimeout(() => {
                this.$q.loading.hide();
              }, 3000);
              clearTimeout(this.checkTimer);
            }
          } else {
            this.status = response.data;
            this.$q.loading.show({
              message: this.status,
              boxClass: "bg-grey-2 text-secondary",
              spinnerColor: "secondary",
            });
          }
          if (this.status == "FINISHED") {
            setTimeout(() => {
              // Check if there is an error message
              if (this.errorMessage) {
                console.log("Error Message...")

                this.$q.notify({
                  message: "This software has been analyzed already.",
                  caption: "Do you want to use the existing results or start a new analysis?",
                  position: "center",
                  color: "secondary",
                  icon: "info",
                  timeout: 0,
                  classes: "custom-notification",
                  actions: [
                    {
                      label: "Show Licenses",
                      color: "primary",
                      class: "action-button",
                      title: "Show already analyzed licenses",
                      handler: () => {
                        this.foundLincences();

                      },

                    },

                    {
                      label: "Reanalyze",
                      color: "primary",
                      class: "action-button",
                      title: "Initiate a new analysis",

                      handler: () => {
                        this.reanalyze();
                      },
                    },

                    {
                      label: 'Dismiss',
                      color: 'primary',
                      class: "action-button",
                      title: "Close",
                      handler: () => { /* ... */ }
                    },

                  ],
                });
              } else {
                // Display a success notification
                this.$q.notify({
                  message: "Your code has been successfully uploaded.",
                  caption: "You may now access the list of found licenses.",
                  position: "center",
                  icon: "check",
                  color: '#1A8917',
                  timeout: 0,
                  classes: "pos-custom-notification",
                  actions: [
                    {
                      label: "Show Licenses",
                      class: "pos-action-button",
                      color: "accent",
                      title: "Show found licenses",
                      handler: () => {
                        this.foundLincences();
                      },
                    },
                    {
                      label: 'Dismiss',
                      class: "pos-action-button",
                      color: "accent",
                      title: "Close",
                      handler: () => { /* ... */ }
                    },
                  ],
                });
              }
            }, 2000);
          }
        });
    },
    uploadZipFile() {
      this.$parent.$emit(
        "uploadZipFile",
        this.file.name,
        this.file,
      );
      this.loading = true;
      this.ready();
      this.$q.loading.show({
        message: "Searching for Licenses",
        boxClass: "bg-grey-2 text-secondary",
        spinnerColor: "secondary",
      });
    },
    ready() {
      this.checkTimer = setInterval(() => {
        this.getStatus();
      }, 5000);
    },
    foundLincences() {
      this.showDiv1 = !this.showDiv1;
      axios
        .get(
          this.engineURL + "/api/v1/software/" +
          this.generateSoftwareid() +
          "/licenses"
        )

        .then((response) => {

          this.licenses = response.data;
          console.log("Licenses from backend:", this.licenses);
          this.checkboxSelection = Object.fromEntries(
            Object.keys(this.licenses).map((license) => [license, true])
          );
        })
        .catch((error) => {
          console.error("Error fetching licenses:", error);
        });
    },

    async compatibleLicenses() {
      const selectedLicenses = Object.keys(this.checkboxSelection).filter(
        (license) => this.checkboxSelection[license]
      );
      console.log("changedetailedCompatibleLicensesId fromLR", selectedLicenses);
      this.$emit("changedetailedCompatibleLicensesId", selectedLicenses);
      //  Ensure selectedLicenses is an array
      const licensesArray = Array.isArray(selectedLicenses)
        ? selectedLicenses
        : [selectedLicenses];
      this.$emit("getCompatibleLicenses", licensesArray);
      this.$router.push("/compatibleLicenses");
    },

    reanalyze() {
      // Delete the software with the specified software-id
      axios
        .delete(
          this.engineURL + "/api/v1/software/" + this.generateSoftwareid()
        )
        .then(() => {
          console.log("the software is deleted", this.generateSoftwareid());
          this.submitForm();
        })
        .catch((error) => {
          console.error("Error deleting software:", error);
        });
    },
  },

  watch: {
    dependencyLicenses: {
      handler(newVal) {
        this.selectedLicenseIds = [];
        newVal.forEach(row => {
          if (row.selected) {
            this.selectedLicenseIds.push(...row.license_id);
          }
        });
      },
      deep: true
    }
  }
};
</script>

<style scoped>
.custom-content {
  padding-bottom: 6rem;
  padding-top: 1rem;
}

.center-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 60vh;
  border: 2px dotted #004191;
  max-width: 1000px;
  margin: 20px auto 0;
}

.form-container {
  max-width: 700px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.btn {
  text-transform: capitalize;
  background-color: #1A8917;
  text-transform: capitalize;
  color: white;
}
</style>
