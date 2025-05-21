<template>
  <div class="h-full flex flex-col p-4">
    <h1 class="text-2xl font-bold mb-4 pb-4 text-gray-500">Drones Available</h1>

    <div class="flex items-center gap-2 mb-4 pb-4">
      <input
        v-model="searchQuery"
        type="text"
        placeholder="Search by sit_name..."
        class="border border-gray-300 rounded px-3 py-2 w-full text-gray-500"
      />
    </div>

    <div class="flex-1 overflow-y-auto">
      <div v-if="filteredDrones.length === 0" class="text-gray-500">
        No drones found.
      </div>
      <div v-else class="grid grid-cols-1 gap-4">
        <div
          v-for="drone in filteredDrones"
          :key="drone.dev_id"
          class="rounded-xl shadow-md border border-gray-200 bg-white p-4"
        >
          <h2 class="text-lg font-semibold text-blue-600">
            {{ drone.dev_name }}
          </h2>
          <p class="text-sm text-gray-600">Serial: {{ drone.dev_sn }}</p>
          <p class="text-sm text-gray-600">Site: {{ drone.sit_name }}</p>
          <p class="text-sm text-gray-600">Details: {{ drone.det_text }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from "vue";
import droneData from "@/assets/data/dronesList.json";

const drones = ref(droneData);
const searchQuery = ref("");

const filteredDrones = computed(() => {
  const query = searchQuery.value.trim().toLowerCase();
  return drones.value.filter((d) => d.sit_name.toLowerCase().includes(query));
});
</script>
