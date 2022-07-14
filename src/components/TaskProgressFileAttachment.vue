<template>
  <div id="root">
    <div v-for="(uploadedItem, index) in uploadedItems" :key="uploadedItem.fileId" class="uploaded-files">
      <div class="uploaded-file-item">({{index}}) id:{{uploadedItem.fileId}}</div>
    </div>
    <input ref="filepond"
           type="file"
           class="filepond"
           name="filepond"
           multiple
           data-allow-reorder="true"
           data-max-file-size="3MB"
           data-max-files="3">
  </div>
</template>

<script>
import * as FilePond from 'filepond';

// Import FilePond styles
import "filepond/dist/filepond.min.css";
import "filepond-plugin-image-preview/dist/filepond-plugin-image-preview.min.css";

// Import image preview and file type validation plugins
import FilePondPluginFileValidateType from "filepond-plugin-file-validate-type";
import FilePondPluginImagePreview from "filepond-plugin-image-preview";
import FilePondPluginFileMetadata from "filepond-plugin-file-metadata";


export default {
  components: {},
  name: "TaskProgressFileAttachment",
  props: {
    taskProgressId: {
      type: Number,
      required: true,
    },
  },
  data: function () {
    return {
      uploadedItems: [],
      filePond: null
    };
  },
  mounted: function () {
    FilePond.registerPlugin(
        FilePondPluginFileValidateType,
        FilePondPluginImagePreview,
        FilePondPluginFileMetadata
    );

    this.filePond = new FilePond.create(
        this.$refs.filepond, {
          fileMetadataObject: {
            taskProgressId: this.taskProgressId
          },
          server: {
            // eslint-disable-next-line no-unused-vars
            process: (fieldName, file, metadata, load, error, progress, abort, transfer, options) => {
              const filepondRequest = new XMLHttpRequest();
              const filepondFormData = new FormData();


              // must perform this asynchronously, don't put await, or the abort functions won't work
              this.doFetch(`
                  mutation {
                    createTaskProgressFileUploadLink(input: {name: "${file.name}", mime_type: "${file.type}", task_progress_id: ${metadata.taskProgressId}}){
                      secure_upload_url
                      additional_s3_security_fields
                      file_id
                    }
                  }
              `)
                  .then(res => res.json())
                  .then(response => {
                    const { secure_upload_url: secureUploadURL, additional_s3_security_fields: additionalFieldsStr, file_id: fileId } = response.data.createTaskProgressFileUploadLink;
                    console.log("[RESPONSE] secureUploadURL: " + secureUploadURL)
                    console.log("[RESPONSE] additionalFields: " + additionalFieldsStr)
                    console.log("[RESPONSE] fileId: " + fileId)

                    const fields = JSON.parse(additionalFieldsStr)
                    for(const field in fields) {
                      filepondFormData.append(field, fields[field]);
                    }

                    filepondFormData.append("file", file);
                    filepondRequest.upload.onprogress = function (e) {
                      progress(e.lengthComputable, e.loaded, e.total);
                    };
                    filepondRequest.open("POST", secureUploadURL);
                    filepondRequest.onload = function () {
                      load(`${fileId}`);
                    };
                    filepondRequest.send(filepondFormData);
                  });

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
    this.filePond.on('processfile', this.handleProcessFile.bind(this))
  },
  methods: {
    doFetch: (query) => {
      return fetch("http://localhost:3003/graphql", {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': 'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjMwMX0.GMAjPGEKLE66zl49aZ6a7V134Syw2DjraRx7huwZS1o'
        },
        body: JSON.stringify({query})
      })
    },
    handleProcessFile: function(error, fileItem) {
      console.log("[handleProcessFile] serverId: " + fileItem.serverId)
      this.uploadedItems.push({fileId: fileItem.serverId})
      this.filePond.removeFile(fileItem)
    },
  },
};
</script>
<style scoped>
.uploaded-files {
  border: 1px solid red;
}
.uploaded-file-item {
  border: 1px solid blue;
}

</style>

