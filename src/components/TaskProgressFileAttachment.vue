<template>
  <div id="root">
    <div v-for="(uploadedItem, index) in uploadedItems" :key="uploadedItem.fileId" class="uploaded-files">
      <div class="uploaded-file-item">
        <div>Index: {{ index }}</div>
        <div>name:{{ uploadedItem.name }}</div>
        <div>file_id:{{ uploadedItem.file_id }}</div>
        <div>mimeType:{{ uploadedItem.mime_type }}</div>
        <div v-if="isImageFile(uploadedItem.mime_type)">
          <img class="image" v-bind:src="uploadedItem.file_url">
        </div>
        <div v-if="!isImageFile(uploadedItem.mime_type)">
          <img class="image" src="@/assets/file.png">
        </div>
        <div>
          <button @click="onDelete(uploadedItem)">Delete</button>
        </div>

      </div>
    </div>
    <div class="base-filepond">
      <input ref="filepond"
             type="file"
             class="filepond"
             name="filepond"
             multiple
             data-allow-reorder="true"
             data-max-file-size="3MB"
             data-max-files="1">
    </div>
  </div>
</template>

<script>
import * as FilePond from 'filepond';

// Import FilePond styles
import "filepond/dist/filepond.min.css";
import "filepond-plugin-image-preview/dist/filepond-plugin-image-preview.min.css";

// Import image preview and file type validation plugins
import FilePondPluginFileValidateType from "filepond-plugin-file-validate-type";
// import FilePondPluginImagePreview from "filepond-plugin-image-preview";
import FilePondPluginFileMetadata from "filepond-plugin-file-metadata";

const doFetch = (query) => {
  return fetch("http://localhost:3003/graphql", {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjMwMX0.GMAjPGEKLE66zl49aZ6a7V134Syw2DjraRx7huwZS1o'
    },
    body: JSON.stringify({query})
  })
}

const initFilePond = (element, taskProgressId) => {
  FilePond.registerPlugin(
      FilePondPluginFileValidateType,
      FilePondPluginFileMetadata
  );
  const retVal = new FilePond.create(
      element, {
        fileMetadataObject: {taskProgressId},
        server: {
          // eslint-disable-next-line no-unused-vars
          process: (fieldName, file, metadata, load, error, progress, abort, transfer, options) => {

            // let's create a new formData request
            const filepondRequest = new XMLHttpRequest();
            const filepondFormData = new FormData();

            // must perform this asynchronously, don't put await, or the abort functions won't work
            doFetch(`
                  mutation {
                    createFileUploadLink(input: {name: "${file.name}", mime_type: "${file.type}"}) {
                      secure_upload_url
                      additional_s3_security_fields
                      file_id,
                      file_instance_id
                    }
                  }
              `)
                .then(res => res.json())
                .then(response => {

                  // extract the important fields secureUploadURL, etc...
                  const {
                    secure_upload_url: secureUploadURL,
                    additional_s3_security_fields: additionalFieldsStr,
                    file_id: fileId,
                    file_instance_id: fileInstanceId
                  } = response.data.createFileUploadLink;
                  console.log("[RESPONSE] secureUploadURL: " + secureUploadURL)
                  console.log("[RESPONSE] additionalFields: " + additionalFieldsStr)
                  console.log("[RESPONSE] fileId: " + fileId)
                  console.log("[RESPONSE] fileInstanceId: " + fileInstanceId)

                  // append the additional fields from amazon to the formData post
                  const fields = JSON.parse(additionalFieldsStr)
                  for (const field in fields) {
                    filepondFormData.append(field, fields[field]);
                  }

                  // of course append the actual file to the formData post
                  filepondFormData.append("file", file);

                  // we listen for upload progress and update filePond widget
                  filepondRequest.upload.onprogress = function (e) {
                    progress(e.lengthComputable, e.loaded, e.total);
                  };

                  // open a POST request to the secureUploadURL
                  filepondRequest.open("POST", secureUploadURL);

                  // we listen for upload completion, where we inform graphql-api
                  // service marking the fileInstance as uploaded
                  filepondRequest.onload = function () {
                    doFetch(`
                      mutation {
                        markFileInstanceUploaded(input: {file_instance_id: ${fileInstanceId}})
                      }
                    `).then(() => load(`${fileId}`))
                        // successfully marked the file uploaded now attach
                        .then(() => {
                            doFetch(`
                              mutation {
                                attachFileToTaskProgress(input: {file_id: ${fileId}, task_progress_id: ${metadata.taskProgressId}})
                              }
                            `).then(res => res.json())
                                .then(response => {
                                  if(response?.data?.attachFileToTaskProgress) {
                                    // all is well
                                    console.log("Success attached file")
                                  } else {
                                    throw "Unable to attach file to TaskProgress"
                                  }
                                })
                        }
                    );
                  };

                  // perform the upload
                  filepondRequest.send(filepondFormData);
                }
            );

            return {
              abort: () => {
                // This function is entered if the user has tapped the cancel button
                filepondRequest.abort();
                // Let FilePond know the request has been cancelled
                abort();
              },
            };

          }
        }
      }
  );

  return retVal;
}


export default {
  components: {},
  name: "TaskProgressFileAttachment",
  props: {
    taskProgressId: {
      type: Number,
      required: true,
    },
  },
  data() {
    return {
      uploadedItems: [],
      filePond: null
    }
  },
  mounted() {
    // initialize filePond
    this.filePond = initFilePond(this.$refs.filepond, this.taskProgressId);

    // attach some handlers
    this.filePond.on('processfile', this.handleProcessFile.bind(this))
    this.filePond.on('error', this.handleError.bind(this))
    this.filePond.on('warning', this.handleWarning.bind(this))

    // fetch initial set of files
    this.doGetFiles(this.taskProgressId)
        .then(files => this.uploadedItems = [...files]);

  },
  methods: {
    isImageFile(mimeType) {
      if (mimeType === 'image/jpeg')
        return true;
      return false;
    },
    /**
     * Fetch files associated with this task progress. Store in data.uploadedItems
     *
     * @param taskProgressId
     * @returns {Promise<*[]>}
     */
    doGetFiles(taskProgressId) {
      if (!taskProgressId) {
        return Promise.resolve(null);
      }
      return doFetch(`
        query {
          getFilesForTaskProgress(input: { task_progress_id: ${taskProgressId} }) {
            files {
              name
              mime_type
              file_url
              file_id
            }
          }
        }
      `
      ).then(res => res.json()).then(response => {
        const files = response?.data?.getFilesForTaskProgress?.files
        return files ? [...files] : []
      }).then(files => this.uploadedItems = [...files]);
    },
    /**
     * Handle on delete file button click.
     *
     * @param file
     */
    onDelete: function (file) {
      // delete file mutation
      doFetch(`
        mutation {
          detachFileFromTaskProgress(input: { file_id: ${file.file_id}, task_progress_id: ${this.taskProgressId} })
        }
      `)
          // re-fetch files
          .then(() =>
              this.doGetFiles(this.taskProgressId).then(files => this.uploadedItems = [...files])
          )
    },

    /**
     * Handle a file was uploaded. Refresh list of attached files.
     *
     * @param error
     * @param fileItem
     */
    handleProcessFile: function (error, fileItem) {
      // re-fetch files
      this.doGetFiles(this.taskProgressId);

      // remove the fileItem from file pond
      this.filePond.removeFile(fileItem)
    },

    /**
     * Let's handle errors
     * @param error
     * @param arg
     */
    handleError: function (error, arg) {
      alert("[handleError] error: " + error + ", " + arg)
    },

    /**
     * Let's handle warnings
     * @param error
     */
    handleWarning: function (error) {
      alert("[handleWarning] error: " + JSON.stringify(error))
    },
  },
};
</script>

<style scoped>
.root {
}

.filepond {
  border: 2px solid red;
}

.image {
  height: auto;
  max-height: 200px;
  /*border: 1px solid pink;*/
}

.uploaded-files {
  /*border: 1px solid red;*/
}

.uploaded-file-item {
  padding: 8px;
  border: 1px solid blue;
}

.base-filepond {
  border: 1px solid red;
  max-width: 100px;
  overflow: hidden;

}
</style>

