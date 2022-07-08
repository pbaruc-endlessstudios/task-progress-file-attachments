<template>
  <div id="app">
    <file-pond
        name="test"
        ref="pond"
        label-idle="Drop files here..."
        v-bind:allow-multiple="true"
        accepted-file-types="image/jpeg, image/png"
        v-bind:files="myFiles"
        v-on:init="handleFilePondInit"

        v-on:addfilestart="handleAddFileStart"
        v-on:addfileprogress="handleFileProgress"
        v-on:addfile="handleAddFile"

        v-on:processfilestart="handleProcessFileStart"
        v-on:processfileprogress="handleProcessFileProgress"
        v-on:processfileabort="handleProcessFileAbort"
        v-on:processfilerevert="handleProcessFileRevert"
        v-on:processfile="handleProcessFile"
        v-on:processfiles="handleProcessFiles"

        v-on:removefile="handleRemoveFile"
        v-on:preparefile="handlePrepareFile"

        v-on:updatefiles="handleUpdateFiles"
        v-on:activefile="handleActivateFile"

        v-on:reorderfiles="handleReorderFiles"
    />
  </div>
</template>

<script>
// Import Vue FilePond
import vueFilePond from "vue-filepond";

// Import FilePond styles
import "filepond/dist/filepond.min.css";

// Import FilePond plugins
// Please note that you need to install these plugins separately

// Import image preview plugin styles
import "filepond-plugin-image-preview/dist/filepond-plugin-image-preview.min.css";

// Import image preview and file type validation plugins
import FilePondPluginFileValidateType from "filepond-plugin-file-validate-type";
import FilePondPluginImagePreview from "filepond-plugin-image-preview";

// Create component
const FilePond = vueFilePond(
    FilePondPluginFileValidateType,
    FilePondPluginImagePreview
);

export default {
  name: "HelloWorld",
  data: function () {
    return { myFiles: ["cat.jpeg"] };
  },
  // mounted: function() {
  //   this.$refs.pond.setOptions({
  //     server: {
  //       // eslint-disable-next-line no-unused-vars
  //       process: (fieldName, file, metadata, load, error, progress, abort, transfer, options) => {
  //         console.log("here")
  //       }
  //     }
  //   })
  // },
  methods: {
    handleFilePondInit: function () {
      console.log("FilePond has initialized");

      // FilePond instance methods are available on `this.$refs.pond`
    },
    // handleFilePondProcessFileStart: function(args) {
    //   console.log("FilePond processFileStart: " + JSON.stringify(args));
    // },
    // handleFilePondProcessFile: function(args) {
    //   console.log("FilePond processFile: " + JSON.stringify(args));
    // },
    handleAddFileStart: function(file) {
      console.log("[handleAddFileStart] " + file)
    },
    handleFileProgress: function(file, progress) {
      console.log("[handleFileProgress] " + JSON.stringify({file, progress}, null, 4))
    },
    handleAddFile: function(error, file) {
      console.log("[handleAddFile] " + JSON.stringify({error, file}, null, 4))
    },
    // v-on:processfilestart="handleProcessFileStart"
    handleProcessFileStart: function(file) {
      console.log("[handleProcessFileStart] " + JSON.stringify({file}, null, 4))
    },
    // v-on:processfileprogress="handleProcessFileProgress"
    handleProcessFileProgress: function(file, progress) {
      console.log("[handleProcessFileProgress] " + JSON.stringify({file, progress}, null, 4))
    },
    // v-on:processfileabort="handleProcessFileAbort"
    handleProcessFileAbort: function(file) {
      console.log("[handleProcessFileAbort] " + JSON.stringify({file}, null, 4))
    },
    // v-on:processfilerevert="handleProcessFileAbort"
    handleProcessFileRevert: function(file) {
      console.log("[handleProcessFileRevert] " + JSON.stringify({file}, null, 4))
    },
    // v-on:processfile="handleProcessFile"
    handleProcessFile: function(file) {
      console.log("[handleProcessFile] " + JSON.stringify({file}, null, 4))
    },
    // v-on:processfiles="handleProcessFiles"
    handleProcessFiles: function() {
      console.log("[handleProcessFiles]")
    },
    // v-on:removefile="handleRemoveFile"
    handleRemoveFile: function(error, file) {
      console.log("[handleRemoveFile] " + JSON.stringify({error, file}, null, 4))
    },
    // v-on:preparefile="handlePrepareFile"
    handlePrepareFile: function(file, output) {
      console.log("[handlePrepareFile] " + JSON.stringify({file, output}, null, 4))
    },
    // v-on:updatefiles="handleUpdateFiles"
    handleUpdateFiles: function(files) {
      console.log("[handleUpdateFiles] " + JSON.stringify({files}, null, 4))
    },
    // v-on:activefile="handleActivateFile"
    handleActivateFile: function(files) {
      console.log("[handleActivateFile] " + JSON.stringify({files}, null, 4))
    },
    // v-on:reorderfiles="handleReorderFiles"
    handleReorderFiles: function(files, origin, target) {
      console.log("[handleActivateFile] " + JSON.stringify({files, origin, target}, null, 4))
    },

  },
  components: {
    FilePond,
  },
};
</script>
