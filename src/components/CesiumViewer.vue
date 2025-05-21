<template>
  <div class="w-full h-screen flex flex-row">
    <div id="cesiumContainer" class="relative w-full h-full">
      <div
        class="absolute bottom-4 left-1/2 transform -translate-x-1/2 bg-black bg-opacity-80 px-4 py-2 rounded shadow flex gap-4 items-center z-10 bg-white"
      >
        <button
          @click="prevStep"
          class="flex items-center gap-1 text-gray-700 cursor-pointer hover:[background-color:#2b7efe] hover:[color:white] rounded px-2 py-1"
        >
          <svg
            class="w-5 h-5"
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M15 19l-7-7 7-7"
            />
          </svg>
          Previous
        </button>

        <button
          v-if="!playing"
          @click="play"
          class="flex items-center gap-1 text-gray-700 cursor-pointer hover:[background-color:#2b7efe] hover:[color:white] rounded px-2 py-1"
        >
          Play
          <svg
            class="w-5 h-5"
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M14.752 11.168l-5.197-3.028A1 1 0 008 9.028v5.944a1 1 0 001.555.832l5.197-3.028a1 1 0 000-1.664z"
            />
          </svg>
        </button>

        <button
          v-else
          @click="pause"
          class="flex items-center gap-1 text-gray-700 cursor-pointer hover:[background-color:#2b7efe] hover:[color:white] rounded px-2 py-1"
        >
          <svg
            class="w-5 h-5"
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M10 9v6m4-6v6"
            />
          </svg>
          Pause
        </button>

        <button
          @click="nextStep"
          class="flex items-center gap-1 text-gray-700 cursor-pointer hover:[background-color:#2b7efe] hover:[color:white] rounded px-2 py-1"
        >
          Next
          <svg
            class="w-5 h-5"
            fill="none"
            stroke="currentColor"
            viewBox="0 0 24 24"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M9 5l7 7-7 7"
            />
          </svg>
        </button>
      </div>
    </div>
    <div
      class="absolute bottom-4 right-4 bg-black bg-opacity-80 px-4 py-2 rounded shadow flex gap-4 items-center z-10 bg-white"
    >
      <ul>
        <li v-for="(step, index) in flightSteps" :key="step.timestamp">
          <a
            href="#"
            @click.prevent="goToStep(index)"
            class="text-gray-700 cursor-pointer hover:bg-gray-200 rounded px-2 py-1"
          >
            {{ step.timestamp }}
          </a>
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import * as Cesium from "cesium";
import flightData from "@/assets/data/flightData.json";

Cesium.Ion.defaultAccessToken = import.meta.env.VITE_CESIUM_TOKEN;

export default {
  data() {
    return {
      flightSteps: [],
      currentIndex: 0,
      playing: false,
      intervalId: null,
      viewer: null,
      droneEntity: null,
    };
  },
  async mounted() {
    // Initialise le terrainProvider ici
    this.terrainProvider = await Cesium.CesiumTerrainProvider.fromIonAssetId(1);

    // Initialise le viewer en utilisant le terrainProvider récupéré
    this.viewer = new Cesium.Viewer("cesiumContainer", {
      terrainProvider: this.terrainProvider,
      baseLayerPicker: false,
      timeline: false,
      animation: false,
      fullscreenButton: false,
    });
    // Transforme ton JSON en flightSteps
    const flightRecordsObj = flightData.flight_records;
    this.flightSteps = Object.entries(flightRecordsObj)
      .map(([timestamp, data]) => ({ timestamp, data }))
      .sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp));

    // Affiche la trajectoire complète
    await this.showFlightPath();

    // Crée l'entité drone (exemple simple : un point avec orientation)
    this.droneEntity = this.viewer.entities.add({
      position: Cesium.Cartesian3.fromDegrees(0, 0, 0),
      point: { pixelSize: 10, color: Cesium.Color.BLUE },
      orientation: Cesium.Quaternion.IDENTITY,
    });

    // Affiche la première étape
    this.updateDronePosition();
  },
  methods: {
    headingPitchRollToQuaternion(
      heading,
      pitch,
      roll,
      longitude,
      latitude,
      height
    ) {
      const hpr = new Cesium.HeadingPitchRoll(
        Cesium.Math.toRadians(heading),
        Cesium.Math.toRadians(pitch),
        Cesium.Math.toRadians(roll)
      );
      return Cesium.Transforms.headingPitchRollQuaternion(
        Cesium.Cartesian3.fromDegrees(longitude, latitude, height),
        hpr
      );
    },
    async showFlightPath() {
      // Crée un tableau Cartographique des positions sans hauteur initiale
      const cartographics = this.flightSteps.map((step) =>
        Cesium.Cartographic.fromDegrees(step.data.longitude, step.data.latitude)
      );

      // Récupère la hauteur terrain la plus précise pour chaque point
      const updatedPositions = await Cesium.sampleTerrainMostDetailed(
        this.terrainProvider,
        cartographics
      );

      // Compose la liste des positions avec altitude ajustée (terrain + hauteur drone)
      const positions = updatedPositions.map((pos, i) => {
        const groundHeight = pos.height ?? 0;
        const droneHeight = this.flightSteps[i].data.height ?? 0;
        return Cesium.Cartesian3.fromDegrees(
          Cesium.Math.toDegrees(pos.longitude),
          Cesium.Math.toDegrees(pos.latitude),
          groundHeight + droneHeight
        );
      });

      // Ajoute une entité polyline représentant la trajectoire
      this.viewer.entities.add({
        name: "Flight Path",
        polyline: {
          positions: positions,
          width: 3,
          material: new Cesium.PolylineDashMaterialProperty({
            color: Cesium.Color.WHITE,
            dashPattern: 255, // valeur par défaut, tu peux ajuster
          }),
          clampToGround: false,
        },
      });

      // Centre la caméra sur la trajectoire
      this.viewer.zoomTo(this.viewer.entities);
    },
    async updateDronePosition() {
      const step = this.flightSteps[this.currentIndex];
      const { latitude, longitude, height: droneHeight } = step.data;
      const { attitude_head, attitude_pitch, attitude_roll } = step.data;

      // Obtenir la hauteur du terrain à cette position
      const cartographic = Cesium.Cartographic.fromDegrees(longitude, latitude);
      const updatedPositions = await Cesium.sampleTerrainMostDetailed(
        this.terrainProvider,
        [cartographic]
      );

      const groundHeight = updatedPositions[0]?.height ?? 0;
      const trueHeight = groundHeight + (droneHeight ?? 0);

      const position = Cesium.Cartesian3.fromDegrees(
        longitude,
        latitude,
        trueHeight
      );
      const orientation = this.headingPitchRollToQuaternion(
        attitude_head,
        attitude_pitch,
        attitude_roll,
        longitude,
        latitude,
        trueHeight
      );

      this.droneEntity.position = position;
      this.droneEntity.orientation = orientation;
    },
    prevStep() {
      if (this.currentIndex > 0) this.currentIndex--;
      this.updateDronePosition();
    },
    nextStep() {
      if (this.currentIndex < this.flightSteps.length - 1) this.currentIndex++;
      this.updateDronePosition();
    },
    play() {
      if (this.playing) return;
      this.playing = true;
      this.intervalId = setInterval(() => {
        if (this.currentIndex < this.flightSteps.length - 1) {
          this.currentIndex++;
          this.updateDronePosition();
        } else {
          this.pause();
        }
      }, 1000);
    },
    pause() {
      this.playing = false;
      clearInterval(this.intervalId);
    },
    goToStep(index) {
      this.currentIndex = index;
      this.updateDronePosition();
    },
  },
};
</script>
