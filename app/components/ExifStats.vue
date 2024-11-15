<template>
  <div class="p-6 flex flex-col gap-6">
    <div class="flex flex-col gap-4">
      <h1 class="text-2xl font-bold">Focal Length and Aperture Statistics</h1>
      <p class="text-sm text-gray-700">
        Select a folder to process images. If your browser does not support directory selection,
        a file input dialog will open instead. All processing is done locally in your browser; no data is sent to any server.
      </p>

      <UButton @click="selectFolder" :disabled="isLoading" class="w-fit">
        {{ isLoading ? 'Processing...' : 'Select Folder' }}
      </UButton>

      <div v-if="isLoading">
        <p class="text-sm text-gray-700">Processing photos: {{ currentProgress }}/{{ totalPhotos }}</p>
        <UProgress :value="progress"/>
      </div>
    </div>

    <div v-show="!isLoading && totalPhotos > 0" class="flex flex-col gap-6">
      <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
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
      <div
        v-if="Object.keys(cameraCounts).length > 1 || Object.keys(lensCounts).length > 1"
        class="flex flex-col md:flex-row gap-4"
      >
        <UFormField label="Filter by camera" v-if="Object.keys(cameraCounts).length > 1">
          <USelect
              class="min-w-64"
              v-model="selectedCamera"
              :items="[{label: 'All cameras', value: null}, ...Object.keys(cameraCounts).map((camera) => ({ label: camera, value: camera }))]"
          >
          </USelect>
        </UFormField>

        <UFormField label="Filter by lens" v-if="Object.keys(lensCounts).length > 1">
          <USelect
              class="min-w-64"
              v-model="selectedLens"
              :items="[{label: 'All lenses', value: null}, ...Object.keys(lensCounts).map((lens) => ({ label: lens, value: lens }))]"
          >
          </USelect>
        </UFormField>
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
import {ref, computed, nextTick, watch} from 'vue'
import * as exifr from 'exifr'
import {Chart, BarController, BarElement, CategoryScale, LinearScale, Title, Tooltip} from 'chart.js'

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

// Helper Functions

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

const processDirectory = async (dirHandle) => {
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
  isLoading.value = false
}

const processFiles = async (files) => {
  focalLengths.value = []
  apertures.value = []
  allData.value = []
  totalPhotos.value = files.length
  currentProgress.value = 0
  isLoading.value = true
  cameraCounts.value = {}
  lensCounts.value = {}
  selectedCamera.value = null
  selectedLens.value = null

  const promises = files.map(async (file) => {
    await extractExifData(file)
    currentProgress.value++
  })

  await Promise.all(promises)
  await renderCharts()
  isLoading.value = false
}

const lensColorMap = {}

const getRandomColor = () => {
  const r = Math.floor(Math.random() * 255)
  const g = Math.floor(Math.random() * 255)
  const b = Math.floor(Math.random() * 255)
  return `rgba(${r}, ${g}, ${b}, 0.6)`
}

const getLensColor = (lens) => {
  if (!lensColorMap[lens]) {
    lensColorMap[lens] = getRandomColor()
  }
  return lensColorMap[lens]
}


const selectFolder = async () => {
  if (window.showDirectoryPicker) {
    try {
      const dirHandle = await window.showDirectoryPicker()
      await processDirectory(dirHandle)
    } catch (error) {
      console.error('Error accessing directory:', error)
    }
  } else {
    const input = document.createElement('input')
    input.type = 'file'
    input.accept = 'image/*'
    input.multiple = true
    input.onchange = async (event) => {
      const files = Array.from(event.target.files)
      await processFiles(files)
    }
    input.click()
  }
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
  let file

  // Determine if fileHandle is a File or FileSystemFileHandle
  if (fileHandle.getFile) {
    file = await fileHandle.getFile()
  } else {
    file = fileHandle // It's already a File
  }

  try {
    const metadata = await exifr.parse(file, ['FocalLength', 'FNumber', 'Make', 'Model', 'LensMake', 'LensModel'])

    if (metadata) {
      const camera = `${metadata.Make || 'Unknown'} ${metadata.Model || 'camera'}`
      const lens = `${metadata.LensMake || 'Unknown'} ${metadata.LensModel || 'lens'}`

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
          title: {display: true, text: 'Focal Length (mm)'}
        },
        y: {
          stacked: true,
          title: {display: true, text: 'Frequency'},
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
          title: {display: true, text: 'Aperture (f-stop)'}
        },
        y: {
          stacked: true,
          title: {display: true, text: 'Frequency'},
          beginAtZero: true
        }
      }
    }
  })
}

// Progress bar calculation
const progress = computed(() => (currentProgress.value / totalPhotos.value) * 100)
</script>
