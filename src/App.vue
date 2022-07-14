<template>
  <img alt="Vue logo" src="./assets/logo.png">
  <TaskProgressFileAttachment task-progress-id="800"/>
</template>

<script>
import TaskProgressFileAttachment from "@/components/TaskProgressFileAttachment";

export default {
  name: 'App',
  components: {
    TaskProgressFileAttachment,
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
