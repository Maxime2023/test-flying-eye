<template>
  <div class="w-full h-screen flex flex-row">
    <!-- Sidebar (Drones List) -->
    <div class="w-1/5 h-full overflow-y-auto bg-white shadow-md p-4">
      <DronesList />
    </div>

    <!-- Cesium Container -->
    <div id="cesiumContainer" class="w-4/5 h-full"></div>
  </div>
</template>

<script setup>
import { onMounted } from "vue";
import * as Cesium from "cesium";
import DronesList from "./DronesList.vue";

// Set the Cesium Ion access token
Cesium.Ion.defaultAccessToken = import.meta.env.VITE_CESIUM_TOKEN;

onMounted(async () => {
  const terrainProvider = await Cesium.CesiumTerrainProvider.fromIonAssetId(1);

  const viewer = new Cesium.Viewer("cesiumContainer", {
    terrainProvider,
    baseLayerPicker: false,
    timeline: false,
    animation: false,
    fullscreenButton: false,
  });

  viewer.entities.add({
    position: Cesium.Cartesian3.fromDegrees(-75.59777, 40.03883),
    point: { pixelSize: 10, color: Cesium.Color.RED },
  });
});
</script>

<style scoped>
@import "cesium/Build/Cesium/Widgets/widgets.css";

/* Make sure Cesium doesn't stretch to full window */
#cesiumContainer canvas {
  display: block;
}
</style>
