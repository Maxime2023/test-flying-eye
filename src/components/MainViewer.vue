<template>
  <div class="w-full h-screen flex flex-row">
    <div class="w-1/5 h-full overflow-y-auto bg-white shadow-md p-4 border-r">
      <DronesList />
    </div>
    <CesiumViewer />
  </div>
</template>

<script setup>
import DronesList from "./DronesList.vue";
import CesiumViewer from "./CesiumViewer.vue";
</script>

<style scoped>
@import "cesium/Build/Cesium/Widgets/widgets.css";

/* Make sure Cesium doesn't stretch to full window */
#cesiumContainer canvas {
  display: block;
}
</style>
