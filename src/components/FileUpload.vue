<template>
  <div id="app">
    <div v-for="(uploadedItem, index) in uploadedItems" :key="uploadedItem.filename" class="uploaded-files">
      <div class="uploaded-file-item">({{index}}) {{uploadedItem.filename}}</div>
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


export default {
  name: "FileUpload",
  data: function () {
    return {
      uploadedItems: [],
      filePond: null
    };
  },
  mounted: function () {
    FilePond.registerPlugin(
        FilePondPluginFileValidateType,
        FilePondPluginImagePreview
    );
    this.filePond = new FilePond.create(
        this.$refs.filepond, {
          server: {
            // eslint-disable-next-line no-unused-vars
            process: (fieldName, file, metadata, load, error, progress, abort, transfer, options) => {
              const filepondRequest = new XMLHttpRequest();
              const filepondFormData = new FormData();

              // must perform this asynchronously, don't put await, or the abort functions won't work
              this.doFetch(`
                  mutation {
                    test(input: {fileName: "${file.name}", contentType: "${file.type}"}){
                      secureUploadURL
                      fields
                    }
                  }
              `)
                  .then(res => res.json())
                  .then(response => {
                    const { secureUploadURL, fields: fieldsStr } = response.data.test;
                    console.log("[RESPONSE] secureUploadURL: " + secureUploadURL)
                    console.log("[RESPONSE] fields: " + fieldsStr)

                    const fields = JSON.parse(fieldsStr)
                    for(const field in fields) {
                      filepondFormData.append(field, fields[field]);
                    }

                    filepondFormData.append("file", file);
                    filepondRequest.upload.onprogress = function (e) {
                      progress(e.lengthComputable, e.loaded, e.total);
                    };
                    filepondRequest.open("POST", secureUploadURL);
                    filepondRequest.onload = function () {
                      load(`${fields.key}`);
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
    // handleFilePondInit: function () {
    //   console.log("FilePond has initialized");
    //
    //   // FilePond instance methods are available on `this.$refs.pond`
    // },
    // // handleFilePondProcessFileStart: function(args) {
    // //   console.log("FilePond processFileStart: " + JSON.stringify(args));
    // // },
    // // handleFilePondProcessFile: function(args) {
    // //   console.log("FilePond processFile: " + JSON.stringify(args));
    // // },
    // handleAddFileStart: function(file) {
    //   console.log("[handleAddFileStart] " + file)
    // },
    // handleFileProgress: function(file, progress) {
    //   console.log("[handleFileProgress] " + JSON.stringify({file, progress}, null, 4))
    // },
    // handleAddFile: function(error, file) {
    //   console.log("[handleAddFile] " + JSON.stringify({error, file}, null, 4))
    // },
    // // v-on:processfilestart="handleProcessFileStart"
    // handleProcessFileStart: function(file) {
    //   console.log("[handleProcessFileStart] " + JSON.stringify({file}, null, 4))
    // },
    // // v-on:processfileprogress="handleProcessFileProgress"
    // handleProcessFileProgress: function(file, progress) {
    //   console.log("[handleProcessFileProgress] " + JSON.stringify({file, progress}, null, 4))
    // },
    // // v-on:processfileabort="handleProcessFileAbort"
    // handleProcessFileAbort: function(file) {
    //   console.log("[handleProcessFileAbort] " + JSON.stringify({file}, null, 4))
    // },
    // // v-on:processfilerevert="handleProcessFileAbort"
    // handleProcessFileRevert: function(file) {
    //   console.log("[handleProcessFileRevert] " + JSON.stringify({file}, null, 4))
    // },
    handleProcessFile: function(error, fileItem) {
      console.log("[handleProcessFile] " + JSON.stringify({
        name: fileItem.filename,
        ext: fileItem.fileExtension,
        size: fileItem.fileSize
      }, null, 4))
      this.uploadedItems.push(fileItem)
      this.filePond.removeFile(fileItem)
    },
    // // v-on:processfiles="handleProcessFiles"
    // handleProcessFiles: function() {
    //   console.log("[handleProcessFiles]")
    // },
    // // v-on:removefile="handleRemoveFile"
    // handleRemoveFile: function(error, file) {
    //   console.log("[handleRemoveFile] " + JSON.stringify({error, file}, null, 4))
    // },
    // // v-on:preparefile="handlePrepareFile"
    // handlePrepareFile: function(file, output) {
    //   console.log("[handlePrepareFile] " + JSON.stringify({file, output}, null, 4))
    // },
    // // v-on:updatefiles="handleUpdateFiles"
    // handleUpdateFiles: function(files) {
    //   console.log("[handleUpdateFiles] " + JSON.stringify({files}, null, 4))
    // },
    // // v-on:activefile="handleActivateFile"
    // handleActivateFile: function(files) {
    //   console.log("[handleActivateFile] " + JSON.stringify({files}, null, 4))
    // },
    // // v-on:reorderfiles="handleReorderFiles"
    // handleReorderFiles: function(files, origin, target) {
    //   console.log("[handleActivateFile] " + JSON.stringify({files, origin, target}, null, 4))
    // },

  },
  components: {},
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

