<template>
  <div class="p-6">
    <h1 class="text-2xl font-bold mb-4">Focal Length and Aperture Statistics</h1>
    <UButton @click="selectFolder" :disabled="isLoading" class="mb-6">
      {{ isLoading ? 'Processing...' : 'Select Folder' }}
    </UButton>
    <div v-if="isLoading" class="mb-6">
      <p class="text-sm text-gray-700">Processing photos: {{ currentProgress }}/{{ totalPhotos }}</p>
      <UProgress :value="progress" />
    </div>
    <div v-show="!isLoading && totalPhotos > 0">
      <p class="text-sm text-gray-700 mb-4">Total Photos Processed: {{ totalPhotos }}</p>
      <div class="grid grid-cols-1 md:grid-cols-2 gap-6 mb-6">
        <UCard class="p-4">
          <h3 class="text-lg font-semibold mb-3">Camera Statistics</h3>
          <ul class="list-disc pl-4">
            <li v-for="(count, camera) in cameraCounts" :key="camera">
              <span class="font-medium">{{ camera }}</span>: {{ count }} photos
            </li>
          </ul>
        </UCard>
        <UCard class="p-4">
          <h3 class="text-lg font-semibold mb-3">Lens Statistics</h3>
          <ul class="list-disc pl-4">
            <li v-for="(count, lens) in lensCounts" :key="lens">
              <span class="font-medium">{{ lens }}</span>: {{ count }} photos
            </li>
          </ul>
        </UCard>
      </div>

      <!-- Filters -->
      <div class="flex flex-col md:flex-row gap-4 mb-6">
        <div>
          <label for="camera-filter" class="block text-sm font-medium text-gray-700">Filter by Camera</label>
          <select
              id="camera-filter"
              v-model="selectedCamera"
              class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50"
          >
            <option :value="null">All Cameras</option>
            <option v-for="(count, camera) in cameraCounts" :key="camera" :value="camera">
              {{ camera }}
            </option>
          </select>
        </div>
        <div>
          <label for="lens-filter" class="block text-sm font-medium text-gray-700">Filter by Lens</label>
          <select
              id="lens-filter"
              v-model="selectedLens"
              class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50"
          >
            <option :value="null">All Lenses</option>
            <option v-for="(count, lens) in lensCounts" :key="lens" :value="lens">
              {{ lens }}
            </option>
          </select>
        </div>
      </div>

      <!-- Graphs -->
      <canvas ref="focalLengthChart" width="400" height="200" class="mb-6"></canvas>
      <canvas ref="apertureChart" width="400" height="200"></canvas>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, nextTick, watch } from 'vue'
import * as exifr from 'exifr'
import { Chart, BarController, BarElement, CategoryScale, LinearScale, Title, Tooltip } from 'chart.js'

// Register Chart.js components
Chart.register(BarController, BarElement, CategoryScale, LinearScale, Title, Tooltip)

// Data stores
const focalLengths = ref([])
const apertures = ref([])
const allData = ref([]) // Store all data for filtering
const totalPhotos = ref(0)
const currentProgress = ref(0)
const isLoading = ref(false)

// Chart instances and refs
const focalLengthChart = ref(null)
const apertureChart = ref(null)
let focalLengthChartInstance = null
let apertureChartInstance = null

// Statistics and filters
const cameraCounts = ref({})
const lensCounts = ref({})
const selectedCamera = ref(null)
const selectedLens = ref(null)

// Filtered data
const filteredData = computed(() => {
  return allData.value.filter(item => {
    const matchesCamera = selectedCamera.value ? item.camera === selectedCamera.value : true
    const matchesLens = selectedLens.value ? item.lens === selectedLens.value : true
    return matchesCamera && matchesLens
  })
})

// Gather all files from the directory and subdirectories
const gatherAllFiles = async (directoryHandle) => {
  const fileHandles = []

  const traverseDirectory = async (directory) => {
    for await (const [name, entry] of directory.entries()) {
      if (entry.kind === 'file' && /\.(jpg|jpeg|png)$/i.test(name)) {
        fileHandles.push(entry)
      } else if (entry.kind === 'directory') {
        await traverseDirectory(entry)
      }
    }
  }

  await traverseDirectory(directoryHandle)
  return fileHandles
}

// Render graphs
const renderCharts = async () => {
  await nextTick()

  const filteredFocalLengths = filteredData.value.map(item => Math.round(item.focalLength))
  const filteredApertures = filteredData.value.map(item => parseFloat(item.aperture))

  // Destroy existing charts
  if (focalLengthChartInstance) {
    focalLengthChartInstance.destroy()
    focalLengthChartInstance = null
  }
  if (apertureChartInstance) {
    apertureChartInstance.destroy()
    apertureChartInstance = null
  }

  // Create new charts
  focalLengthChartInstance = createFocalLengthChart(filteredFocalLengths)
  apertureChartInstance = createApertureChart(filteredApertures)
}

// Watch filters to update graphs
watch([selectedCamera, selectedLens], () => {
  renderCharts()
})

// Folder selection
const selectFolder = async () => {
  try {
    const dirHandle = await window.showDirectoryPicker()
    focalLengths.value = []
    apertures.value = []
    allData.value = []
    totalPhotos.value = 0
    currentProgress.value = 0
    isLoading.value = true
    cameraCounts.value = {}
    lensCounts.value = {}
    selectedCamera.value = null
    selectedLens.value = null

    const fileHandles = await gatherAllFiles(dirHandle)
    totalPhotos.value = fileHandles.length

    await processFilesConcurrently(fileHandles)
    await renderCharts()
  } catch (error) {
    console.error('Error accessing directory:', error)
  } finally {
    isLoading.value = false
  }
}

// Process files concurrently
const processFilesConcurrently = async (fileHandles) => {
  const promises = fileHandles.map(async (fileHandle) => {
    await extractExifData(fileHandle)
    currentProgress.value++
  })

  await Promise.all(promises)
}

// Extract EXIF data
const extractExifData = async (fileHandle) => {
  const file = await fileHandle.getFile()
  try {
    const metadata = await exifr.parse(file, ['FocalLength', 'FNumber', 'Make', 'Model', 'LensModel'])

    if (metadata) {
      const camera = `${metadata.Make || 'Unknown'} ${metadata.Model || 'Model'}`
      const lens = metadata.LensModel || 'Unknown Lens'

      allData.value.push({
        focalLength: metadata.FocalLength,
        aperture: metadata.FNumber,
        camera,
        lens
      })

      if (cameraCounts.value[camera]) {
        cameraCounts.value[camera]++
      } else {
        cameraCounts.value[camera] = 1
      }

      if (lensCounts.value[lens]) {
        lensCounts.value[lens]++
      } else {
        lensCounts.value[lens] = 1
      }
    }
  } catch (error) {
    console.warn(`Error reading EXIF for ${file.name}:`, error)
  }
}

// Create charts
const createFocalLengthChart = (filteredFocalLengths) => {
  const focalLengthCounts = filteredFocalLengths.reduce((counts, length) => {
    counts[length] = (counts[length] || 0) + 1
    return counts
  }, {})

  const sortedFocalLengths = Object.entries(focalLengthCounts).sort((a, b) => a[0] - b[0])
  const labels = sortedFocalLengths.map(entry => entry[0])
  const data = sortedFocalLengths.map(entry => entry[1])

  return new Chart(focalLengthChart.value.getContext('2d'), {
    type: 'bar',
    data: {
      labels,
      datasets: [{
        label: 'Focal Length Frequency',
        data,
        backgroundColor: 'rgba(54, 162, 235, 0.6)',
        borderColor: 'rgba(54, 162, 235, 1)',
        borderWidth: 1
      }]
    },
    options: {
      scales: {
        y: { beginAtZero: true, title: { display: true, text: 'Frequency' } },
        x: { title: { display: true, text: 'Focal Length (mm)' } }
      }
    }
  })
}

const createApertureChart = (filteredApertures) => {
  const apertureCounts = filteredApertures.reduce((counts, aperture) => {
    counts[aperture] = (counts[aperture] || 0) + 1
    return counts
  }, {})

  const sortedApertures = Object.entries(apertureCounts).sort((a, b) => parseFloat(a[0]) - parseFloat(b[0]))
  const labels = sortedApertures.map(entry => entry[0])
  const data = sortedApertures.map(entry => entry[1])

  return new Chart(apertureChart.value.getContext('2d'), {
    type: 'bar',
    data: {
      labels,
      datasets: [{
        label: 'Aperture Frequency (f-stop)',
        data,
        backgroundColor: 'rgba(255, 159, 64, 0.6)',
        borderColor: 'rgba(255, 159, 64, 1)',
        borderWidth: 1
      }]
    },
    options: {
      scales: {
        y: { beginAtZero: true, title: { display: true, text: 'Frequency' } },
        x: { title: { display: true, text: 'Aperture (f-stop)' } }
      }
    }
  })
}

// Progress bar calculation
const progress = computed(() => (currentProgress.value / totalPhotos.value) * 100)
</script>

<style>
.loading-indicator {
  margin-top: 20px;
}
</style>
