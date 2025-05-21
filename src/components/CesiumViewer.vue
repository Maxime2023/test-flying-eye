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
      class="absolute bottom-4 right-4 bg-black bg-opacity-80 px-4 py-2 rounded shadow flex gap-4 items-center z-10 bg-white w-1/4 h-1/2 overflow-y-auto"
    >
      <ul class="space-y-3 h-full overflow-y-auto">
        <li
          v-for="(step, index) in flightSteps"
          :key="step.timestamp"
          @click.prevent="goToStep(index)"
          :class="[
            'cursor-pointer rounded-lg shadow-md p-4 transition-colors duration-200',
            index === currentIndex
              ? 'bg-blue-600 text-white shadow-blue-500'
              : 'bg-white hover:bg-gray-100 text-gray-800',
          ]"
        >
          <div class="flex justify-between items-center mb-1">
            <span class="font-semibold text-lg">
              {{ formatTimestamp(step.timestamp) }}
            </span>
            <span
              class="text-xs font-medium rounded-full px-2 py-0.5 bg-blue-400 text-white"
            >
              Step {{ index + 1 }}
            </span>
          </div>
          <div class="text-sm space-x-4 flex flex-wrap gap-2">
            <div>
              <strong>Height:</strong>
              {{ step.data.height?.toFixed(1) ?? "-" }} m
            </div>
            <div>
              <strong>Battery:</strong>
              {{ step.data.battery?.capacity_percent ?? "-" }}%
            </div>
            <div>
              <strong>Speed:</strong>
              {{ step.data.horizontal_speed?.toFixed(1) ?? "-" }} m/s
            </div>
            <div>
              <strong>Pitch:</strong>
              {{ step.data.attitude_pitch?.toFixed(1) ?? "-" }}°
            </div>
            <div>
              <strong>Roll:</strong>
              {{ step.data.attitude_roll?.toFixed(1) ?? "-" }}°
            </div>
            <div>
              <strong>Heading:</strong>
              {{ step.data.attitude_head?.toFixed(1) ?? "-" }}°
            </div>
          </div>
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
    this.terrainProvider = await Cesium.CesiumTerrainProvider.fromIonAssetId(1);

    this.viewer = new Cesium.Viewer("cesiumContainer", {
      terrainProvider: this.terrainProvider,
      baseLayerPicker: false,
      timeline: false,
      animation: false,
      fullscreenButton: false,
    });

    const flightRecordsObj = flightData.flight_records;
    this.flightSteps = Object.entries(flightRecordsObj)
      .map(([timestamp, data]) => ({ timestamp, data }))
      .sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp));

    await this.showFlightPath();
    await this.addVerticalLines();

    this.droneEntity = this.viewer.entities.add({
      position: Cesium.Cartesian3.fromDegrees(0, 0, 0),
      point: { pixelSize: 10, color: Cesium.Color.BLUE },
      orientation: Cesium.Quaternion.IDENTITY,
    });

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
      const cartographics = this.flightSteps.map((step) =>
        Cesium.Cartographic.fromDegrees(step.data.longitude, step.data.latitude)
      );

      const updatedPositions = await Cesium.sampleTerrainMostDetailed(
        this.terrainProvider,
        cartographics
      );

      const positions = updatedPositions.map((pos, i) => {
        const groundHeight = pos.height ?? 0;
        const droneHeight = this.flightSteps[i].data.height ?? 0;
        return Cesium.Cartesian3.fromDegrees(
          Cesium.Math.toDegrees(pos.longitude),
          Cesium.Math.toDegrees(pos.latitude),
          groundHeight + droneHeight
        );
      });

      const flightPathEntity = this.viewer.entities.add({
        name: "Flight Path",
        polyline: {
          positions: positions,
          width: 3,
          material: new Cesium.PolylineArrowMaterialProperty(
            Cesium.Color.WHITE
          ),
          clampToGround: false,
        },
      });

      await this.viewer.flyTo(flightPathEntity);
    },
    drawAxes(position, orientation) {
      if (this.axesEntities) {
        this.axesEntities.forEach((entity) => {
          this.viewer.entities.remove(entity);
        });
      }
      this.axesEntities = []; // reset tableau

      const length = 50;
      const origin = position;

      const transform = Cesium.Matrix4.fromTranslationQuaternionRotationScale(
        origin,
        orientation,
        new Cesium.Cartesian3(1, 1, 1)
      );

      this.axesEntities.push(
        this.viewer.entities.add({
          polyline: {
            positions: [
              origin,
              Cesium.Matrix4.multiplyByPoint(
                transform,
                new Cesium.Cartesian3(length, 0, 0),
                new Cesium.Cartesian3()
              ),
            ],
            width: 3,
            material: Cesium.Color.RED,
          },
        })
      );

      this.axesEntities.push(
        this.viewer.entities.add({
          polyline: {
            positions: [
              origin,
              Cesium.Matrix4.multiplyByPoint(
                transform,
                new Cesium.Cartesian3(0, length, 0),
                new Cesium.Cartesian3()
              ),
            ],
            width: 3,
            material: Cesium.Color.GREEN,
          },
        })
      );

      this.axesEntities.push(
        this.viewer.entities.add({
          polyline: {
            positions: [
              origin,
              Cesium.Matrix4.multiplyByPoint(
                transform,
                new Cesium.Cartesian3(0, 0, length),
                new Cesium.Cartesian3()
              ),
            ],
            width: 3,
            material: Cesium.Color.BLUE,
          },
        })
      );
    },

    async addVerticalLines() {
      // Remove previous vertical lines and labels if any
      if (this.verticalLineEntities) {
        this.verticalLineEntities.forEach((entity) =>
          this.viewer.entities.remove(entity)
        );
      }
      if (this.labelEntities) {
        this.labelEntities.forEach((entity) =>
          this.viewer.entities.remove(entity)
        );
      }

      this.verticalLineEntities = [];
      this.labelEntities = [];

      for (let i = 0; i < this.flightSteps.length; i++) {
        const step = this.flightSteps[i];
        const { longitude, latitude, height: droneHeight } = step.data;

        const cartographic = Cesium.Cartographic.fromDegrees(
          longitude,
          latitude
        );
        const updatedPositions = await Cesium.sampleTerrainMostDetailed(
          this.terrainProvider,
          [cartographic]
        );
        const groundHeight = updatedPositions[0]?.height ?? 0;

        const dronePosition = Cesium.Cartesian3.fromDegrees(
          longitude,
          latitude,
          groundHeight + (droneHeight ?? 0)
        );
        const groundPosition = Cesium.Cartesian3.fromDegrees(
          longitude,
          latitude,
          groundHeight
        );

        // Add vertical line (arrow material if you want)
        const verticalLineEntity = this.viewer.entities.add({
          polyline: {
            positions: [groundPosition, dronePosition],
            width: 4,
            material: new Cesium.PolylineDashMaterialProperty({
              color: Cesium.Color.WHITE,
              dashPattern: 255,
            }),
          },
        });
        this.verticalLineEntities.push(verticalLineEntity);

        const labelPosition = Cesium.Cartesian3.fromDegrees(
          longitude,
          latitude,
          groundHeight + (droneHeight ?? 0) + 5
        );

        const labelEntity = this.viewer.entities.add({
          position: labelPosition,
          label: {
            text: `${i + 1}`, // Label text as 1, 2, 3...
            font: "16px sans-serif",
            fillColor: Cesium.Color.WHITE,
            outlineColor: Cesium.Color.BLACK,
            outlineWidth: 2,
            style: Cesium.LabelStyle.FILL_AND_OUTLINE,
            verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
            heightReference: Cesium.HeightReference.NONE,
          },
        });
        this.labelEntities.push(labelEntity);
      }
    },
    formatTimestamp(timestamp) {
      const date = new Date(timestamp);
      return date.toLocaleString(undefined, {
        year: "numeric",
        month: "short",
        day: "numeric",
        hour: "2-digit",
        minute: "2-digit",
        second: "2-digit",
      });
    },
    createFrustum(position, orientation) {
      if (this.frustumEntity) {
        this.viewer.entities.remove(this.frustumEntity);
      }

      this.drawAxes(position, orientation);
    },

    async updateDronePosition() {
      const step = this.flightSteps[this.currentIndex];
      const { latitude, longitude, height: droneHeight } = step.data;
      const { attitude_head, attitude_pitch, attitude_roll } = step.data;

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

      this.createFrustum(position, orientation);
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

      if (this.currentIndex >= this.flightSteps.length - 1) {
        this.currentIndex = 0;
        this.updateDronePosition();
      }

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
