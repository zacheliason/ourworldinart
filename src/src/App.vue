<template>
  <div id="layout">
    <Map />
  </div>
</template>

<script>
  import Map from './components/Map.vue';
  import '../node_modules/mapbox-gl/dist/mapbox-gl.css';

  export default {
    components: {
      Map
    }
  };
</script>

<style>
  #layout {
    flex: 1;
    display: flex;
    position: relative;
  }
</style>