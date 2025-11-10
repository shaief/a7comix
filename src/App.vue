<template>
  <div class="container">
    <h1>📄 A4 to A7 Document Splitter</h1>
    <p class="subtitle">Upload an A4 scanned document and split it into 8 A7-sized JPG images</p>

    <!-- Upload Area -->
    <div
      v-if="!selectedFile"
      class="upload-area"
      :class="{ dragging: isDragging }"
      @click="triggerFileInput"
      @dragover.prevent="isDragging = true"
      @dragleave.prevent="isDragging = false"
      @drop.prevent="handleDrop"
    >
      <div class="upload-icon">📎</div>
      <div class="upload-text">Click to upload or drag and drop</div>
      <div class="upload-hint">Supports JPG, PNG, PDF and other image formats</div>
      <input
        ref="fileInput"
        type="file"
        class="file-input"
        accept="image/*,application/pdf"
        @change="handleFileSelect"
      >
    </div>

    <!-- File Info & Actions -->
    <div v-if="selectedFile" class="file-info">
      <h3>Selected File:</h3>
      <div class="file-name">{{ selectedFile.name }}</div>

      <div class="button-group">
        <button
          class="btn btn-primary"
          @click="processDocument"
          :disabled="processing"
        >
          {{ processing ? 'Processing...' : 'Split into A7 Images' }}
        </button>
        <button
          class="btn btn-secondary"
          @click="reset"
          :disabled="processing"
        >
          Choose Different File
        </button>
      </div>

      <!-- Progress Bar -->
      <div v-if="processing" class="progress-bar">
        <div class="progress-fill" :style="{ width: progress + '%' }"></div>
      </div>
    </div>

    <!-- Status Messages -->
    <div v-if="statusMessage" :class="['status-message', `status-${statusType}`]">
      {{ statusMessage }}
    </div>

    <!-- Download Button -->
    <div v-if="processedImages.length > 0" class="button-group">
      <button class="btn btn-primary" @click="downloadAll">
        Download All Images ({{ processedImages.length }})
      </button>
    </div>

    <!-- Preview Grid -->
    <div v-if="processedImages.length > 0" class="preview-grid">
      <div v-for="(img, index) in processedImages" :key="index" class="preview-item">
        <img :src="img.dataUrl" :alt="img.filename">
        <div class="preview-label">{{ img.filename }}</div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import * as pdfjsLib from 'pdfjs-dist'

// State
const selectedFile = ref(null)
const fileInput = ref(null)
const isDragging = ref(false)
const processing = ref(false)
const progress = ref(0)
const processedImages = ref([])
const statusMessage = ref('')
const statusType = ref('info')

// Initialize PDF.js worker
onMounted(() => {
  pdfjsLib.GlobalWorkerOptions.workerSrc = `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.379/pdf.worker.min.js`
})

// File handling
const triggerFileInput = () => {
  fileInput.value.click()
}

const handleFileSelect = (event) => {
  const file = event.target.files[0]
  if (file) {
    selectedFile.value = file
    processedImages.value = []
    statusMessage.value = ''
  }
}

const handleDrop = (event) => {
  isDragging.value = false
  const file = event.dataTransfer.files[0]
  if (file) {
    selectedFile.value = file
    processedImages.value = []
    statusMessage.value = ''
  }
}

const reset = () => {
  selectedFile.value = null
  processedImages.value = []
  statusMessage.value = ''
  progress.value = 0
  if (fileInput.value) {
    fileInput.value.value = ''
  }
}

// Image processing
const processDocument = async () => {
  if (!selectedFile.value) return

  processing.value = true
  progress.value = 0
  processedImages.value = []
  statusMessage.value = 'Processing document...'
  statusType.value = 'info'

  try {
    const file = selectedFile.value
    const isPdf = file.type === 'application/pdf'

    if (isPdf) {
      await processPdf(file)
    } else {
      await processImage(file)
    }

    statusMessage.value = `Successfully generated ${processedImages.value.length} A7 images!`
    statusType.value = 'success'
  } catch (error) {
    console.error('Error processing document:', error)
    statusMessage.value = `Error: ${error.message}`
    statusType.value = 'error'
  } finally {
    processing.value = false
    progress.value = 100
  }
}

const processPdf = async (file) => {
  const arrayBuffer = await file.arrayBuffer()
  const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise

  const baseName = file.name.replace(/\.\w+$/, '')
  let imageCounter = 1

  for (let pageNum = 1; pageNum <= pdf.numPages; pageNum++) {
    progress.value = (pageNum / pdf.numPages) * 50

    const page = await pdf.getPage(pageNum)
    const canvas = document.createElement('canvas')
    const context = canvas.getContext('2d')

    // Render at high resolution
    const viewport = page.getViewport({ scale: 2.0 })
    canvas.width = viewport.width
    canvas.height = viewport.height

    await page.render({
      canvasContext: context,
      viewport: viewport
    }).promise

    // Split this page into A7 pieces
    const pieces = await splitIntoA7(canvas, baseName, imageCounter)
    processedImages.value.push(...pieces)
    imageCounter += pieces.length

    progress.value = 50 + (pageNum / pdf.numPages) * 50
  }
}

const processImage = async (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()

    reader.onload = async (e) => {
      try {
        progress.value = 30

        const img = new Image()
        img.onload = async () => {
          const canvas = document.createElement('canvas')
          canvas.width = img.width
          canvas.height = img.height

          const ctx = canvas.getContext('2d')
          ctx.drawImage(img, 0, 0)

          progress.value = 50

          const baseName = file.name.replace(/\.\w+$/, '')
          const pieces = await splitIntoA7(canvas, baseName, 1)
          processedImages.value = pieces

          progress.value = 100
          resolve()
        }

        img.onerror = () => reject(new Error('Failed to load image'))
        img.src = e.target.result
      } catch (error) {
        reject(error)
      }
    }

    reader.onerror = () => reject(new Error('Failed to read file'))
    reader.readAsDataURL(file)
  })
}

const splitIntoA7 = async (canvas, baseName, startIndex) => {
  const pieces = []
  const width = canvas.width
  const height = canvas.height

  // A4 to A7: split into 2 columns × 4 rows = 8 pieces
  const cols = 2
  const rows = 4
  const pieceWidth = width / cols
  const pieceHeight = height / rows

  for (let row = 0; row < rows; row++) {
    for (let col = 0; col < cols; col++) {
      const pieceCanvas = document.createElement('canvas')
      pieceCanvas.width = pieceWidth
      pieceCanvas.height = pieceHeight

      const ctx = pieceCanvas.getContext('2d')

      // Extract piece from original canvas
      ctx.drawImage(
        canvas,
        col * pieceWidth, // source x
        row * pieceHeight, // source y
        pieceWidth, // source width
        pieceHeight, // source height
        0, // dest x
        0, // dest y
        pieceWidth, // dest width
        pieceHeight // dest height
      )

      // Convert to JPG
      const dataUrl = pieceCanvas.toBlob ?
        await new Promise(resolve => {
          pieceCanvas.toBlob(blob => {
            resolve(URL.createObjectURL(blob))
          }, 'image/jpeg', 0.95)
        }) :
        pieceCanvas.toDataURL('image/jpeg', 0.95)

      const pieceIndex = startIndex + (row * cols + col)
      pieces.push({
        filename: `${baseName}_${pieceIndex}.jpg`,
        dataUrl: dataUrl,
        canvas: pieceCanvas
      })
    }
  }

  return pieces
}

// Download functionality
const downloadAll = () => {
  processedImages.value.forEach((img, index) => {
    // Create download link
    const link = document.createElement('a')

    if (img.dataUrl.startsWith('blob:')) {
      link.href = img.dataUrl
    } else {
      // Convert data URL to blob for better download support
      fetch(img.dataUrl)
        .then(res => res.blob())
        .then(blob => {
          const url = URL.createObjectURL(blob)
          link.href = url
          link.download = img.filename
          link.click()
          setTimeout(() => URL.revokeObjectURL(url), 100)
        })
      return
    }

    link.download = img.filename
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)

    // Small delay between downloads to avoid browser blocking
    if (index < processedImages.value.length - 1) {
      setTimeout(() => {}, 100)
    }
  })

  statusMessage.value = 'Download started for all images!'
  statusType.value = 'success'
}
</script>
