<template>
  <div class="p-6 bg-gray-100 min-h-screen">
    <h1 class="text-3xl font-extrabold text-blue-700 mb-6">🛰️ Create My Asset List</h1>

    <!-- Filters Section - One Line -->
    <div class="flex flex-wrap md:flex-nowrap gap-4 mb-6 items-end">
      <!-- Search Input -->
      <div class="flex-1 min-w-[200px]">
        <label class="block mb-1 font-semibold text-gray-700">Search</label>
        <input
          v-model="searchQuery"
          @keyup.enter="searchTable"
          placeholder="Name or NORAD ID"
          class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring focus:ring-blue-400"
        />
      </div>

      <!-- Object Type Dropdown -->
      <div class="relative flex-1 min-w-[200px]">
        <label class="block mb-1 font-semibold text-gray-700">Object Type</label>
        <div
          @click="toggleDropdown"
          class="w-full bg-white border border-gray-300 rounded-lg px-4 py-2 cursor-pointer flex justify-between items-center hover:shadow"
        >
          <span class="truncate">
            {{ selectedObjectTypes.length ? selectedObjectTypes.join(', ') : 'Select Types' }}
          </span>
          <svg class="w-4 h-4 text-gray-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M19 9l-7 7-7-7"
            />
          </svg>
        </div>
        <div
          v-if="dropdownOpen"
          class="absolute z-10 bg-white border mt-1 rounded-lg shadow max-h-60 overflow-y-auto w-full"
        >
          <div
            v-for="type in objectTypes"
            :key="type"
            class="p-2 flex items-center hover:bg-gray-50"
          >
            <input type="checkbox" :value="type" v-model="selectedObjectTypes" class="mr-2" />
            <label class="text-sm text-gray-800">{{ type }}</label>
          </div>
        </div>
      </div>

      <!-- Orbit Code Dropdown -->
      <div class="relative flex-1 min-w-[200px]">
        <label class="block mb-1 font-semibold text-gray-700">Orbit Code</label>
        <div
          @click="toggleOrbitDropdown"
          class="w-full bg-white border border-gray-300 rounded-lg px-4 py-2 cursor-pointer flex justify-between items-center hover:shadow"
        >
          <span class="truncate">
            {{ selectedOrbitCodes.length ? selectedOrbitCodes.join(', ') : 'Select Orbits' }}
          </span>
          <svg class="w-4 h-4 text-gray-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M19 9l-7 7-7-7"
            />
          </svg>
        </div>
        <div
          v-if="orbitDropdownOpen"
          class="absolute z-10 bg-white border mt-1 rounded-lg shadow max-h-60 overflow-y-auto w-full"
        >
          <div
            v-for="code in orbitCodes"
            :key="code"
            class="p-2 flex items-center hover:bg-gray-50"
          >
            <input type="checkbox" :value="code" v-model="selectedOrbitCodes" class="mr-2" />
            <label class="text-sm text-gray-800">{{ code }}</label>
          </div>
        </div>
      </div>

      <!-- Apply Filters Button -->
      <div class="mt-6">
        <button
          @click="applyFilters"
          class="bg-blue-600 hover:bg-blue-700 text-white font-semibold px-6 py-2 rounded-lg transition whitespace-nowrap"
        >
          Apply Filters
        </button>
      </div>
    </div>

    <!-- Selected Count -->
    <div
      v-if="selectedRows.length"
      class="mb-4 flex justify-between items-center bg-blue-50 p-3 rounded-lg border border-blue-200"
    >
      <span class="text-blue-800 font-semibold">Selected: {{ selectedRows.length }} / 10</span>
      <button
        @click="proceedToSelection"
        class="bg-green-600 hover:bg-green-700 text-white font-semibold px-5 py-2 rounded-lg"
      >
        Proceed
      </button>
    </div>

    <!-- Table -->
    <div v-if="loading" class="text-center text-gray-500">🔄 Loading satellites...</div>
    <div v-else-if="error" class="text-center text-red-600">❌ Failed to load satellite data.</div>
    <div
      v-show="!loading && !error"
      id="example-table"
      class="bg-white shadow rounded-lg overflow-hidden mt-4"
    ></div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import axios from 'axios'
import { TabulatorFull as Tabulator } from 'tabulator-tables'
import 'tabulator-tables/dist/css/tabulator.min.css'

const router = useRouter()

const searchQuery = ref('')
const selectedObjectTypes = ref([])
const selectedOrbitCodes = ref([])
const selectedRows = ref([])
const loading = ref(false)
const error = ref(false)
const rawData = ref([])
let table = null
const dropdownOpen = ref(false)

const objectTypes = ['PAYLOAD', 'ROCKET BODY', 'DEBRIS', 'UNKNOWN']
const orbitCodes = [
  'LEO',
  'LEO1',
  'LEO2',
  'LEO3',
  'LEO4',
  'MEO',
  'GEO',
  'HEO',
  'IGO',
  'EGO',
  'NSO',
  'GTO',
  'GHO',
  'HAO',
  'MGO',
  'LMO',
  'UFO',
  'ESO',
  'UNKNOWN',
]

const orbitDropdownOpen = ref(false)
function toggleOrbitDropdown() {
  orbitDropdownOpen.value = !orbitDropdownOpen.value
}

async function fetchSatellitesData() {
  loading.value = true
  error.value = false
  try {
    const params = {
      attributes: 'noradCatId,name,orbitCode,objectType,countryCode,launchDate',
    }
    if (selectedObjectTypes.value.length) {
      params.objectTypes = selectedObjectTypes.value.join(',')
    }

    const res = await axios.get('https://backend.digantara.dev/v1/satellites', { params })
    let data = res.data.data

    // Frontend filtering for orbitCode
    if (selectedOrbitCodes.value.length) {
      data = data.filter((item) => selectedOrbitCodes.value.includes(item.orbitCode))
    }

    rawData.value = data
    initTable()
  } catch (err) {
    console.error(err)
    error.value = true
  } finally {
    loading.value = false
  }
}

function initTable() {
  if (table) {
    table.setData(rawData.value)
    return
  }

  table = new Tabulator('#example-table', {
    data: rawData.value,
    height: '500px',
    layout: 'fitColumns',
    selectable: true,
    selectableCheck: function () {
      return true
    },
    rowSelected: function (row) {
      if (selectedRows.value.length >= 10) {
        row.deselect()
        alert('Maximum 10 selections allowed.')
        return
      }
      selectedRows.value.push(row.getData())
    },
    rowDeselected: function (row) {
      const id = row.getData().noradCatId
      selectedRows.value = selectedRows.value.filter((item) => item.noradCatId !== id)
    },
    columns: [
      {
        title: '',
        formatter: 'rowSelection',
        titleFormatter: 'rowSelection',
        hozAlign: 'center',
        headerSort: false,
        width: 40,
      },
      { title: 'Name', field: 'name', sorter: 'string' },
      { title: 'Norad ID', field: 'noradCatId', sorter: 'number' },
      { title: 'Orbit Code', field: 'orbitCode' },
      { title: 'Object Type', field: 'objectType' },
      { title: 'Country', field: 'countryCode' },
      {
        title: 'Launch Date',
        field: 'launchDate',
        sorter: 'date',
        formatter: (cell) => cell.getValue() || '—',
      },
    ],
  })
}

function applyFilters() {
  selectedRows.value = [] // reset selection
  fetchSatellitesData()
}

function searchTable() {
  const query = searchQuery.value.trim().toLowerCase()
  table.setFilter(
    (data) =>
      data.name?.toLowerCase().includes(query) || data.noradCatId?.toString().includes(query)
  )
}

function proceedToSelection() {
  const selected = selectedRows.value.map(({ name, noradCatId }) => ({ name, noradCatId }))
  localStorage.setItem('selectedSatellites', JSON.stringify(selected))
  router.push('/selected')
}

function toggleDropdown() {
  dropdownOpen.value = !dropdownOpen.value
}

onMounted(fetchSatellitesData)
</script>

<style scoped>
/* Optional enhancements */
select[multiple] {
  scrollbar-width: thin;
}
</style>