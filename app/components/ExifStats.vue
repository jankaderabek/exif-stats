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
        <UCard>
          <h3 class="text-lg font-semibold mb-3">Camera Statistics</h3>
          <ul class="list-disc pl-4">
            <li v-for="(count, camera) in cameraCounts" :key="camera">
              <span class="font-medium">{{ camera }}</span>: {{ count }} photos
            </li>
          </ul>
        </UCard>
        <UCard>
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
      <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <div>
          <canvas ref="focalLengthChart" class="mb-6"></canvas>
        </div>
        <div>
          <canvas ref="apertureChart"></canvas>
        </div>
      </div>
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

const colorPalette = [
  'rgba(91, 192, 222, 0.6)', // Light Blue
  'rgba(240, 173, 78, 0.6)', // Amber
  'rgba(92, 184, 92, 0.6)',  // Green
  'rgba(217, 83, 79, 0.6)',  // Red
  'rgba(155, 89, 182, 0.6)', // Purple
  'rgba(52, 152, 219, 0.6)', // Sky Blue
  'rgba(241, 196, 15, 0.6)', // Yellow
  'rgba(39, 174, 96, 0.6)',  // Emerald
  'rgba(231, 76, 60, 0.6)',  // Alizarin
  'rgba(52, 73, 94, 0.6)',   // Midnight Blue
]

const lensColorMap = {}

const getLensColor = (lens) => {
  if (!lensColorMap[lens]) {
    // Randomly select a color from the palette
    const randomIndex = Math.floor(Math.random() * colorPalette.length)
    lensColorMap[lens] = colorPalette[randomIndex]
  }
  return lensColorMap[lens]
}


// Render graphs
const renderCharts = async () => {
  await nextTick()

  // Filtered data for charts
  const filteredFocalLengths = filteredData.value
  const filteredApertures = filteredData.value

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
const createFocalLengthChart = (filteredData) => {
  const groupedData = {}
  filteredData.forEach(item => {
    const focalLength = Math.round(item.focalLength)
    const lens = item.lens

    if (!groupedData[focalLength]) {
      groupedData[focalLength] = {}
    }
    groupedData[focalLength][lens] = (groupedData[focalLength][lens] || 0) + 1
  })

  const lenses = [...new Set(filteredData.map(item => item.lens))]
  const labels = Object.keys(groupedData).sort((a, b) => a - b)
  const datasets = lenses.map(lens => {
    const data = labels.map(focalLength => groupedData[focalLength]?.[lens] || 0)
    return {
      label: lens,
      data,
      backgroundColor: getLensColor(lens),
    }
  })

  return new Chart(focalLengthChart.value.getContext('2d'), {
    type: 'bar',
    data: {
      labels,
      datasets
    },
    options: {
      plugins: {
        title: {
          display: true,
          text: 'Focal Length Distribution by Lens'
        },
        tooltip: {
          mode: 'index',
          intersect: false
        }
      },
      responsive: true,
      scales: {
        x: {
          stacked: true,
          title: { display: true, text: 'Focal Length (mm)' }
        },
        y: {
          stacked: true,
          title: { display: true, text: 'Frequency' },
          beginAtZero: true
        }
      }
    }
  })
}

const createApertureChart = (filteredData) => {
  const groupedData = {}
  filteredData.forEach(item => {
    const aperture = parseFloat(item.aperture).toFixed(1)
    const lens = item.lens

    if (!groupedData[aperture]) {
      groupedData[aperture] = {}
    }
    groupedData[aperture][lens] = (groupedData[aperture][lens] || 0) + 1
  })

  const lenses = [...new Set(filteredData.map(item => item.lens))]
  const labels = Object.keys(groupedData).sort((a, b) => parseFloat(a) - parseFloat(b))
  const datasets = lenses.map(lens => {
    const data = labels.map(aperture => groupedData[aperture]?.[lens] || 0)
    return {
      label: lens,
      data,
      backgroundColor: getLensColor(lens),
    }
  })

  return new Chart(apertureChart.value.getContext('2d'), {
    type: 'bar',
    data: {
      labels,
      datasets
    },
    options: {
      plugins: {
        title: {
          display: true,
          text: 'Aperture Distribution by Lens'
        },
        tooltip: {
          mode: 'index',
          intersect: false
        }
      },
      responsive: true,
      scales: {
        x: {
          stacked: true,
          title: { display: true, text: 'Aperture (f-stop)' }
        },
        y: {
          stacked: true,
          title: { display: true, text: 'Frequency' },
          beginAtZero: true
        }
      }
    }
  })
}

// Progress bar calculation
const progress = computed(() => (currentProgress.value / totalPhotos.value) * 100)
</script>
