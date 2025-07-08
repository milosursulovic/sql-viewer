<script setup>
import initSqlJs from "sql.js";
import { ref, reactive, onMounted, computed } from "vue";

const db = ref(null);
const SQL = ref(null);
const tables = ref([]);
const query = ref("");
const results = reactive({ columns: [], values: [] });
const error = ref("");
const searchQuery = ref("");
const currentPage = ref(1);
const rowsPerPage = 10;

const filteredValues = computed(() => {
  if (!searchQuery.value) return results.values;
  return results.values.filter((row) =>
    row.some((cell) =>
      cell?.toString().toLowerCase().includes(searchQuery.value.toLowerCase())
    )
  );
});

const paginatedValues = computed(() => {
  const start = (currentPage.value - 1) * rowsPerPage;
  return filteredValues.value.slice(start, start + rowsPerPage);
});

const totalPages = computed(() =>
  Math.ceil(filteredValues.value.length / rowsPerPage)
);

onMounted(async () => {
  SQL.value = await initSqlJs({
    locateFile: (file) => `https://sql.js.org/dist/${file}`,
  });
});

function handleFileUpload(event) {
  const file = event.target.files[0];
  if (!file || !SQL.value) return;

  const reader = new FileReader();
  reader.onload = () => {
    try {
      const uInt8Array = new Uint8Array(reader.result);
      db.value = new SQL.value.Database(uInt8Array);

      const res = db.value.exec(
        "SELECT name FROM sqlite_master WHERE type='table'"
      );
      tables.value = res[0]?.values.map((row) => row[0]) || [];
    } catch (e) {
      error.value = "Failed to load database file.";
      console.error(e);
    }
  };
  reader.readAsArrayBuffer(file);
}

function runQuery() {
  if (!db.value) return;

  try {
    const res = db.value.exec(query.value);
    if (res.length > 0) {
      results.columns = res[0].columns;
      results.values = res[0].values;
      currentPage.value = 1;
      error.value = "";
    } else {
      results.columns = [];
      results.values = [];
      error.value = "No results returned.";
    }
  } catch (e) {
    error.value = e.message;
    console.error(e);
  }
}
</script>

<template>
  <main class="p-6 max-w-6xl mx-auto space-y-6">
    <h1 class="text-3xl font-bold text-blue-800">SQLite Viewer</h1>

    <div class="border p-4 rounded-lg shadow bg-white">
      <label class="block mb-2 font-semibold">Upload SQLite File</label>
      <input
        type="file"
        @change="handleFileUpload"
        accept=".sqlite,.db"
        class="border p-2 rounded w-full"
      />
    </div>

    <div v-if="db" class="border p-4 rounded-lg shadow bg-white space-y-4">
      <div>
        <label class="block font-medium mb-1">Tables</label>
        <select
          class="border px-4 py-2 rounded w-full bg-gray-50"
          @change="query = `SELECT * FROM \`${$event.target.value}\``"
        >
          <option disabled selected>Choose table</option>
          <option v-for="table in tables" :key="table" :value="table">
            {{ table }}
          </option>
        </select>
      </div>

      <div>
        <label class="block font-medium mb-1">SQL Query</label>
        <textarea
          v-model="query"
          class="w-full border p-2 rounded font-mono bg-gray-50"
          rows="3"
        ></textarea>
      </div>

      <button
        @click="runQuery"
        class="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700"
      >
        Execute
      </button>

      <p v-if="error" class="text-red-600 mt-2">{{ error }}</p>
    </div>

    <div
      v-if="results.columns.length > 0"
      class="overflow-x-auto border p-4 rounded-lg shadow bg-white"
    >
      <div class="mb-4 flex justify-between items-center">
        <input
          type="text"
          v-model="searchQuery"
          placeholder="Search..."
          class="border p-2 rounded w-full max-w-xs"
        />
        <div class="ml-4 text-sm text-gray-600">
          Page {{ currentPage }} of {{ totalPages }}
        </div>
      </div>

      <table class="min-w-full text-sm text-left border">
        <thead>
          <tr class="bg-gray-100 border-b">
            <th
              v-for="col in results.columns"
              :key="col"
              class="p-2 font-semibold"
            >
              {{ col }}
            </th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(row, idx) in paginatedValues" :key="idx" class="border-t">
            <td v-for="(cell, i) in row" :key="i" class="p-2 whitespace-nowrap">
              {{ cell }}
            </td>
          </tr>
        </tbody>
      </table>

      <div class="flex justify-between items-center mt-4">
        <button
          :disabled="currentPage <= 1"
          @click="currentPage--"
          class="px-3 py-1 rounded bg-gray-200 hover:bg-gray-300 disabled:opacity-50"
        >
          Previous
        </button>

        <button
          :disabled="currentPage >= totalPages"
          @click="currentPage++"
          class="px-3 py-1 rounded bg-gray-200 hover:bg-gray-300 disabled:opacity-50"
        >
          Next
        </button>
      </div>
    </div>
  </main>
</template>
