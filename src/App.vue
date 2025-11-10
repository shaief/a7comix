<template>
  <div class="container">
    <h1>📄 A4 to A7 Document Splitter</h1>
    <p class="subtitle">Upload an A4 document (portrait or landscape) and automatically split it into 8 properly-sized A7 JPG images</p>

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

    <!-- File Info & Preview -->
    <div v-if="selectedFile && !processedImages.length" class="file-info">
      <h3>Selected File:</h3>
      <div class="file-name">{{ selectedFile.name }}</div>

      <!-- Source Preview -->
      <div v-if="previewUrl" class="source-preview-section">
        <div class="preview-header">
          <h3>Preview:</h3>
          <div v-if="pdfPageCount > 0" class="page-info">
            Page {{ selectedPage }} of {{ pdfPageCount }}
          </div>
        </div>
        <div class="preview-container">
          <canvas
            ref="previewCanvas"
            class="source-preview-canvas"
            :style="{ transform: `rotate(${rotation}deg)` }"
          ></canvas>
        </div>

        <!-- Page Selection Toggle (for PDFs with 2 pages) -->
        <div v-if="pdfPageCount === 2" class="page-selection">
          <h4>Select Page to Process:</h4>
          <div class="page-selection-buttons">
            <button
              class="btn-page-select"
              :class="{ active: selectedPage === 1 }"
              @click="setPage(1)"
              :disabled="processing"
            >
              First Page
            </button>
            <button
              class="btn-page-select"
              :class="{ active: selectedPage === 2 }"
              @click="setPage(2)"
              :disabled="processing"
            >
              Second Page
            </button>
          </div>
        </div>

        <!-- Rotation Controls -->
        <div class="rotation-controls">
          <h4>Rotate Document:</h4>
          <div class="rotation-buttons">
            <button
              class="btn-rotation"
              :class="{ active: rotation === 0 }"
              @click="setRotation(0)"
              :disabled="processing"
            >
              0°
            </button>
            <button
              class="btn-rotation"
              :class="{ active: rotation === 90 }"
              @click="setRotation(90)"
              :disabled="processing"
            >
              90°
            </button>
            <button
              class="btn-rotation"
              :class="{ active: rotation === 180 }"
              @click="setRotation(180)"
              :disabled="processing"
            >
              180°
            </button>
            <button
              class="btn-rotation"
              :class="{ active: rotation === 270 }"
              @click="setRotation(270)"
              :disabled="processing"
            >
              270°
            </button>
          </div>
        </div>
      </div>

      <div class="button-group">
        <button
          class="btn btn-primary"
          @click="processDocument"
          :disabled="processing || !previewUrl"
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
import { ref, onMounted, nextTick } from 'vue'
import * as pdfjsLib from 'pdfjs-dist'

// State
const selectedFile = ref(null)
const fileInput = ref(null)
const previewCanvas = ref(null)
const isDragging = ref(false)
const processing = ref(false)
const progress = ref(0)
const processedImages = ref([])
const statusMessage = ref('')
const statusType = ref('info')
const previewUrl = ref('')
const rotation = ref(0)
const sourceCanvas = ref(null)
const selectedPage = ref(1) // 1 for first page, 2 for second page
const pdfPageCount = ref(0)
const pdfDocument = ref(null)

// Initialize PDF.js worker
onMounted(() => {
  // Use the local worker file from public directory
  pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.mjs'
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
    rotation.value = 0
    selectedPage.value = 1
    pdfPageCount.value = 0
    pdfDocument.value = null
    loadPreview(file)
  }
}

const handleDrop = (event) => {
  isDragging.value = false
  const file = event.dataTransfer.files[0]
  if (file) {
    selectedFile.value = file
    processedImages.value = []
    statusMessage.value = ''
    rotation.value = 0
    selectedPage.value = 1
    pdfPageCount.value = 0
    pdfDocument.value = null
    loadPreview(file)
  }
}

const reset = () => {
  selectedFile.value = null
  processedImages.value = []
  statusMessage.value = ''
  progress.value = 0
  previewUrl.value = ''
  rotation.value = 0
  sourceCanvas.value = null
  selectedPage.value = 1
  pdfPageCount.value = 0
  pdfDocument.value = null
  if (fileInput.value) {
    fileInput.value.value = ''
  }
}

// Rotation control
const setRotation = (degrees) => {
  rotation.value = degrees
}

// Page selection control
const setPage = async (pageNum) => {
  selectedPage.value = pageNum
  if (pdfDocument.value) {
    // Reload preview with new page
    const page = await pdfDocument.value.getPage(pageNum)
    const canvas = document.createElement('canvas')
    const context = canvas.getContext('2d')

    const viewport = page.getViewport({ scale: 2.0 })
    canvas.width = viewport.width
    canvas.height = viewport.height

    await page.render({
      canvasContext: context,
      viewport: viewport
    }).promise

    sourceCanvas.value = canvas
    await drawPreview(canvas)
  }
}

// Load preview
const loadPreview = async (file) => {
  try {
    const isPdf = file.type === 'application/pdf'

    if (isPdf) {
      await loadPdfPreview(file)
    } else {
      await loadImagePreview(file)
    }
  } catch (error) {
    console.error('Error loading preview:', error)
    statusMessage.value = `Error loading preview: ${error.message}`
    statusType.value = 'error'
  }
}

const loadImagePreview = async (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()

    reader.onload = async (e) => {
      const img = new Image()
      img.onload = async () => {
        // Create source canvas
        const canvas = document.createElement('canvas')
        canvas.width = img.width
        canvas.height = img.height
        const ctx = canvas.getContext('2d')
        ctx.drawImage(img, 0, 0)
        sourceCanvas.value = canvas

        // Draw preview
        await drawPreview(canvas)
        resolve()
      }
      img.onerror = () => reject(new Error('Failed to load image'))
      img.src = e.target.result
    }

    reader.onerror = () => reject(new Error('Failed to read file'))
    reader.readAsDataURL(file)
  })
}

const loadPdfPreview = async (file) => {
  const arrayBuffer = await file.arrayBuffer()
  const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise

  // Check page count
  pdfPageCount.value = pdf.numPages

  if (pdf.numPages > 2) {
    statusMessage.value = `⚠️ This PDF has ${pdf.numPages} pages. Only PDFs with 1 or 2 pages are supported. Please upload a different file.`
    statusType.value = 'error'
    selectedFile.value = null
    previewUrl.value = ''
    pdfPageCount.value = 0
    if (fileInput.value) {
      fileInput.value.value = ''
    }
    return
  }

  // Store the PDF document for later use
  pdfDocument.value = pdf

  // Load the selected page (default is page 1)
  const page = await pdf.getPage(selectedPage.value)

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

  sourceCanvas.value = canvas
  await drawPreview(canvas)
}

const drawPreview = async (canvas) => {
  // Set previewUrl first so the canvas element gets rendered
  previewUrl.value = 'loaded'

  // Wait for Vue to render the canvas element
  await nextTick()

  if (!previewCanvas.value) {
    console.error('Preview canvas ref not available')
    return
  }

  const maxWidth = 600
  const maxHeight = 800
  let width = canvas.width
  let height = canvas.height

  // Scale down if needed
  const scale = Math.min(maxWidth / width, maxHeight / height, 1)
  width = width * scale
  height = height * scale

  previewCanvas.value.width = width
  previewCanvas.value.height = height

  const ctx = previewCanvas.value.getContext('2d')
  ctx.drawImage(canvas, 0, 0, width, height)
}

// Apply rotation to canvas
const applyRotation = (sourceCanvas) => {
  if (rotation.value === 0) {
    return sourceCanvas
  }

  const canvas = document.createElement('canvas')
  const ctx = canvas.getContext('2d')

  // For 90 and 270 degrees, swap width and height
  if (rotation.value === 90 || rotation.value === 270) {
    canvas.width = sourceCanvas.height
    canvas.height = sourceCanvas.width
  } else {
    canvas.width = sourceCanvas.width
    canvas.height = sourceCanvas.height
  }

  // Move to center, rotate, then draw
  ctx.translate(canvas.width / 2, canvas.height / 2)
  ctx.rotate((rotation.value * Math.PI) / 180)
  ctx.drawImage(sourceCanvas, -sourceCanvas.width / 2, -sourceCanvas.height / 2)

  return canvas
}

// Image processing
const processDocument = async () => {
  if (!selectedFile.value || !sourceCanvas.value) return

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
      await processImage()
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
  // Use the already loaded PDF document if available
  let pdf = pdfDocument.value
  if (!pdf) {
    const arrayBuffer = await file.arrayBuffer()
    pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise
  }

  const baseName = file.name.replace(/\.\w+$/, '')

  progress.value = 30

  // Process only the selected page
  const page = await pdf.getPage(selectedPage.value)
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

  progress.value = 50

  // Apply rotation
  const rotatedCanvas = applyRotation(canvas)

  // Split this page into A7 pieces
  const pieces = await splitIntoA7(rotatedCanvas, baseName, 1)
  processedImages.value = pieces

  progress.value = 100
}

const processImage = async () => {
  progress.value = 30

  // Use the already loaded source canvas
  const baseName = selectedFile.value.name.replace(/\.\w+$/, '')

  // Apply rotation
  const rotatedCanvas = applyRotation(sourceCanvas.value)

  progress.value = 50

  const pieces = await splitIntoA7(rotatedCanvas, baseName, 1)
  processedImages.value = pieces

  progress.value = 100
}

const splitIntoA7 = async (canvas, baseName, startIndex) => {
  const pieces = []
  const width = canvas.width
  const height = canvas.height

  // Detect orientation: portrait if height > width, landscape otherwise
  const isPortrait = height > width

  // A4 to A7 aspect ratio: A7 is 74mm × 105mm (ratio ≈ 0.7048)
  // For A4 portrait (210×297): split into 2 cols × 4 rows (A7 pieces are landscape 105×74)
  // For A4 landscape (297×210): split into 4 cols × 2 rows (A7 pieces are portrait 74×105)
  let cols, rows, pieceWidth, pieceHeight

  if (isPortrait) {
    // Portrait A4: 2 columns × 4 rows
    cols = 2
    rows = 4
    pieceWidth = width / cols
    pieceHeight = height / rows

    // A7 landscape ratio check (width should be larger than height)
    // Adjust to maintain proper A7 landscape ratio (105:74 = 1.419)
    const targetRatio = 105 / 74  // A7 landscape
    const currentRatio = pieceWidth / pieceHeight

    if (Math.abs(currentRatio - targetRatio) > 0.1) {
      // Adjust dimensions to match A7 ratio better
      const avgSize = (pieceWidth + pieceHeight) / 2
      pieceWidth = avgSize * targetRatio / (1 + targetRatio)
      pieceHeight = avgSize / (1 + targetRatio)
      pieceWidth = width / cols
      pieceHeight = pieceWidth / targetRatio
    }
  } else {
    // Landscape A4: 4 columns × 2 rows
    cols = 4
    rows = 2
    pieceWidth = width / cols
    pieceHeight = height / rows

    // A7 portrait ratio check (height should be larger than width)
    // Adjust to maintain proper A7 portrait ratio (105:74 = 1.419)
    const targetRatio = 105 / 74  // height/width for portrait
    const currentRatio = pieceHeight / pieceWidth

    if (Math.abs(currentRatio - targetRatio) > 0.1) {
      // Adjust dimensions to match A7 ratio better
      pieceWidth = width / cols
      pieceHeight = pieceWidth * targetRatio
    }
  }

  statusMessage.value = `Processing ${isPortrait ? 'portrait' : 'landscape'} document into ${cols}×${rows} A7 pieces...`

  for (let row = 0; row < rows; row++) {
    for (let col = 0; col < cols; col++) {
      const pieceCanvas = document.createElement('canvas')

      // Calculate actual dimensions to extract
      const srcX = col * (width / cols)
      const srcY = row * (height / rows)
      const srcWidth = width / cols
      const srcHeight = height / rows

      // Set canvas to A7 proportions
      pieceCanvas.width = Math.round(srcWidth)
      pieceCanvas.height = Math.round(srcHeight)

      const ctx = pieceCanvas.getContext('2d')
      ctx.imageSmoothingEnabled = true
      ctx.imageSmoothingQuality = 'high'

      // Extract piece from original canvas
      ctx.drawImage(
        canvas,
        srcX, // source x
        srcY, // source y
        srcWidth, // source width
        srcHeight, // source height
        0, // dest x
        0, // dest y
        pieceCanvas.width, // dest width
        pieceCanvas.height // dest height
      )

      // Convert to JPG with high quality
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
