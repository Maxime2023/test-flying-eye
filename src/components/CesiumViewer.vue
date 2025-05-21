<template>
  <div id="cesiumContainer" style="width: 100%; height: 100vh"></div>
</template>

<script setup>
import { onMounted } from "vue";
import * as Cesium from "cesium";

Cesium.Ion.defaultAccessToken = import.meta.env.VITE_CESIUM_TOKEN;

onMounted(async () => {
  const terrainProvider = await Cesium.CesiumTerrainProvider.fromIonAssetId(1);

  const viewer = new Cesium.Viewer("cesiumContainer", {
    terrainProvider,
    baseLayerPicker: false,
    timeline: false,
  });

  viewer.entities.add({
    position: Cesium.Cartesian3.fromDegrees(-75.59777, 40.03883),
    point: { pixelSize: 10, color: Cesium.Color.RED },
  });
});
</script>

<style scoped>
@import "cesium/Build/Cesium/Widgets/widgets.css";
</style>
