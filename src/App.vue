<template>
  <div id="app">
    <load3dModel />
  </div>
</template>

<script>
import load3dModel from "./views/load3dModel.vue";

export default {
  name: "app",
  components: {
    load3dModel,
  },
};
</script>

<style lang="scss">
#app {
  height: 100vh;
  width: 100vw;
}
</style>
