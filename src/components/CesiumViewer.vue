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
const terrainProvider = await Cesium.CesiumTerrainProvider.fromIonAssetId(1);
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
  mounted() {
    // Initialise la viewer Cesium
    this.viewer = new Cesium.Viewer("cesiumContainer", {
      terrainProvider,
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

    // Crée l'entité drone (exemple simple : un point avec orientation)
    this.droneEntity = this.viewer.entities.add({
      position: Cesium.Cartesian3.fromDegrees(0, 0, 0),
      point: { pixelSize: 10, color: Cesium.Color.RED },
      orientation: Cesium.Quaternion.IDENTITY,
      // Pour le cône, on peut ajouter une autre entité ou primitive liée
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
    updateDronePosition() {
      const step = this.flightSteps[this.currentIndex];
      const { latitude, longitude, height } = step.data;
      const { attitude_head, attitude_pitch, attitude_roll } = step.data;

      const position = Cesium.Cartesian3.fromDegrees(
        longitude,
        latitude,
        height
      );
      const orientation = this.headingPitchRollToQuaternion(
        attitude_head,
        attitude_pitch,
        attitude_roll,
        longitude,
        latitude,
        height
      );

      this.droneEntity.position = position;
      this.droneEntity.orientation = orientation;

      // Centrer la caméra sur le drone
      this.viewer.camera.flyTo({ destination: position });
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
