<template>
  <div>
    <h1>Focal Length and Aperture Statistics</h1>
    <button @click="selectFolder">Select Folder</button>
    <div v-if="photoCount > 0">
      <p>Total Photos Processed: {{ photoCount }}</p>
      <canvas ref="focalLengthChart" width="400" height="200" style="margin-top: 30px;"></canvas>
      <canvas ref="apertureChart" width="400" height="200" style="margin-top: 30px;"></canvas>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import * as exifr from 'exifr'
import { Chart, BarController, BarElement, CategoryScale, LinearScale, Title, Tooltip } from 'chart.js'

// Register Chart.js components
Chart.register(BarController, BarElement, CategoryScale, LinearScale, Title, Tooltip)

const focalLengths = ref([])
const apertures = ref([])
const photoCount = ref(0)  // Track the total number of photos processed

const focalLengthChart = ref(null)
const apertureChart = ref(null)

const selectFolder = async () => {
  try {
    const dirHandle = await window.showDirectoryPicker()
    focalLengths.value = []
    apertures.value = []
    photoCount.value = 0  // Reset the photo count

    const fileHandles = await gatherAllFiles(dirHandle)
    await processFilesConcurrently(fileHandles)
    renderCharts()
  } catch (error) {
    console.error('Error accessing directory:', error)
  }
}

const gatherAllFiles = async (directoryHandle) => {
  const fileHandles = []

  const traverseDirectory = async (directory) => {
    const entries = []

    for await (const [name, entry] of directory.entries()) {
      if (entry.kind === 'file' && /\.(jpg|jpeg|png)$/i.test(name)) {
        fileHandles.push(entry)
      } else if (entry.kind === 'directory') {
        entries.push(traverseDirectory(entry))
      }
    }

    await Promise.all(entries)
  }

  await traverseDirectory(directoryHandle)
  return fileHandles
}

const processFilesConcurrently = async (fileHandles) => {
  photoCount.value = fileHandles.length  // Set total photo count
  const promises = fileHandles.map(fileHandle => extractExifData(fileHandle))
  await Promise.all(promises)
}

const extractExifData = async (fileHandle) => {
  const file = await fileHandle.getFile()
  try {
    const metadata = await exifr.parse(file, ['FocalLength', 'FNumber'])

    if (metadata && metadata.FocalLength) {
      focalLengths.value.push(Math.round(metadata.FocalLength))
    }
    if (metadata && metadata.FNumber) {
      apertures.value.push(metadata.FNumber.toFixed(1))
    }
  } catch (error) {
    console.warn(`Error reading EXIF for ${file.name}:`, error)
  }
}

const renderCharts = () => {
  if (focalLengthChart.value && apertureChart.value) {
    createFocalLengthChart()
    createApertureChart()
  }
}

const createFocalLengthChart = () => {
  const focalLengthCounts = focalLengths.value.reduce((counts, length) => {
    counts[length] = (counts[length] || 0) + 1
    return counts
  }, {})

  const sortedFocalLengths = Object.entries(focalLengthCounts).sort((a, b) => a[0] - b[0])
  const labels = sortedFocalLengths.map(entry => entry[0])
  const data = sortedFocalLengths.map(entry => entry[1])

  new Chart(focalLengthChart.value.getContext('2d'), {
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
        y: {beginAtZero: true, title: {display: true, text: 'Frequency'}},
        x: {title: {display: true, text: 'Focal Length (mm)'}}
      }
    }
  })
}

const createApertureChart = () => {
  const apertureCounts = apertures.value.reduce((counts, aperture) => {
    counts[aperture] = (counts[aperture] || 0) + 1
    return counts
  }, {})

  const sortedApertures = Object.entries(apertureCounts).sort((a, b) => parseFloat(a[0]) - parseFloat(b[0]))
  const labels = sortedApertures.map(entry => entry[0])
  const data = sortedApertures.map(entry => entry[1])

  new Chart(apertureChart.value.getContext('2d'), {
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
        y: {beginAtZero: true, title: {display: true, text: 'Frequency'}},
        x: {title: {display: true, text: 'Aperture (f-stop)'}}
      }
    }
  })
}
</script>

<style>
button {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 16px;
}
</style>
