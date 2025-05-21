<template>
  <div>
    <h1 class="text-2xl font-bold mb-4">Drones Available</h1>
    <div class="flex items-center gap-2 mb-4 pt-4">
      <input
        v-model.number="searchLimit"
        class="border border-gray-300 rounded px-3 py-1"
        placeholder="Nombre"
      />
      <button
        @click="fetchDrones"
        class="bg-blue-500 text-white px-4 py-1 rounded hover:bg-blue-600"
      >
        Search
      </button>
    </div>

    <div v-if="loading" class="text-gray-500">Loading...</div>
    <div v-else class="grid grid-cols-1 gap-4">
      <div
        v-for="cat in cats"
        :key="cat.id"
        class="rounded-xl shadow-md overflow-hidden"
      >
        <img
          :src="cat.url"
          :alt="'Chat ' + cat.id"
          class="w-full h-60 object-cover"
        />
        <div class="p-2">
          <p class="text-sm text-gray-700">ID: {{ cat.id }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";

const cats = ref([]);
const loading = ref(true);
const searchLimit = ref(9); // default limit

const fetchDrones = async () => {
  loading.value = true;
  try {
    const res = await fetch(
      `https://api.thecatapi.com/v1/images/search?limit=${searchLimit.value}`
    );
    const data = await res.json();
    cats.value = data;
  } catch (error) {
    console.error("Erreur lors du chargement des chats :", error);
  } finally {
    loading.value = false;
  }
};

onMounted(fetchDrones);
</script>
