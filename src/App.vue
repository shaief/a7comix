<template>
  <div class="container">
    <div class="app-nav">
      <span class="nav-current">📄 A7 Splitter</span>
      <span class="nav-separator">|</span>
      <a href="pager.html" class="nav-link">📖 Pager</a>
    </div>
    <h1>📄 A4 to A7 Document Splitter</h1>
    <p class="subtitle">Upload one A4 page (portrait or landscape), select if it's your first or second page, then split it into 8 properly-sized A7 JPG images</p>

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
      <div class="file-header">
        <div>
          <h3>Selected File:</h3>
          <div class="file-name">{{ selectedFile.name }}</div>
        </div>
        <button
          class="btn btn-secondary btn-small"
          @click="reset"
          :disabled="processing"
        >
          Choose Different File
        </button>
      </div>

      <!-- Source Preview -->
      <div v-if="previewUrl" class="source-preview-section">
        <div class="preview-header">
          <h3>Preview:</h3>
          <div v-if="pdfPageCount > 1" class="page-info">
            Viewing: Page {{ selectedPage }} of {{ pdfPageCount }}
          </div>
        </div>
        <div class="preview-container">
          <canvas
            ref="previewCanvas"
            class="source-preview-canvas"
          ></canvas>
        </div>

        <!-- Modern Settings Controls -->
        <div class="settings-panel">
          <!-- Page Position Switch -->
          <div class="setting-card">
            <div class="setting-header">
              <span class="setting-icon">📄</span>
              <span class="setting-title">Page Position</span>
            </div>
            <div class="switch-container">
              <span class="switch-label" :class="{ active: selectedPage === 1 }">First Page</span>
              <label class="switch">
                <input
                  type="checkbox"
                  :checked="selectedPage === 2"
                  @change="setPagePosition(selectedPage === 1 ? 2 : 1)"
                  :disabled="processing"
                >
                <span class="slider"></span>
              </label>
              <span class="switch-label" :class="{ active: selectedPage === 2 }">Second Page</span>
            </div>
          </div>

          <!-- Rotation Controls -->
          <div class="setting-card">
            <div class="setting-header">
              <span class="setting-icon">🔄</span>
              <span class="setting-title">Rotation</span>
            </div>
            <div class="rotation-grid">
              <button
                class="rotation-btn"
                :class="{ active: rotation === 0 }"
                @click="setRotation(0)"
                :disabled="processing"
                title="No rotation"
              >
                <span class="rotation-icon">↑</span>
                <span class="rotation-degree">0°</span>
              </button>
              <button
                class="rotation-btn"
                :class="{ active: rotation === 90 }"
                @click="setRotation(90)"
                :disabled="processing"
                title="Rotate 90° clockwise"
              >
                <span class="rotation-icon">→</span>
                <span class="rotation-degree">90°</span>
              </button>
              <button
                class="rotation-btn"
                :class="{ active: rotation === 180 }"
                @click="setRotation(180)"
                :disabled="processing"
                title="Rotate 180°"
              >
                <span class="rotation-icon">↓</span>
                <span class="rotation-degree">180°</span>
              </button>
              <button
                class="rotation-btn"
                :class="{ active: rotation === 270 }"
                @click="setRotation(270)"
                :disabled="processing"
                title="Rotate 270° clockwise"
              >
                <span class="rotation-icon">←</span>
                <span class="rotation-degree">270°</span>
              </button>
            </div>
          </div>

          <!-- Numbering Mode Controls -->
          <div class="setting-card">
            <div class="setting-header">
              <span class="setting-icon">🔢</span>
              <span class="setting-title">Numbering Order</span>
            </div>
            <div class="chip-group">
              <button
                class="chip"
                :class="{ active: orderingMode === 'standard' }"
                @click="setOrderingMode('standard')"
                :disabled="processing"
              >
                Standard
              </button>
              <button
                class="chip"
                :class="{ active: orderingMode === 'rtl' }"
                @click="setOrderingMode('rtl')"
                :disabled="processing"
              >
                RTL Booklet
              </button>
              <button
                class="chip"
                :class="{ active: orderingMode === 'ltr' }"
                @click="setOrderingMode('ltr')"
                :disabled="processing"
              >
                LTR Booklet
              </button>
            </div>
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

    <!-- Download and Reset Buttons -->
    <div v-if="processedImages.length > 0" class="button-group">
      <button class="btn btn-primary" @click="downloadAllAsZip">
        Download as ZIP ({{ processedImages.length }} images)
      </button>
      <button class="btn btn-secondary" @click="reset">
        Upload Another Document
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
import JSZip from 'jszip'

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
// Store PDF document in plain variable to avoid Vue reactivity breaking PDF.js private fields
let pdfDocument = null
const orderingMode = ref('rtl') // 'standard', 'rtl', 'ltr'

// Initialize PDF.js worker
onMounted(() => {
  // Use the local worker file from public directory
  // import.meta.env.BASE_URL accounts for GitHub Pages base path
  pdfjsLib.GlobalWorkerOptions.workerSrc = `${import.meta.env.BASE_URL}pdf.worker.mjs`
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
    pdfDocument = null
    orderingMode.value = 'rtl'
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
    pdfDocument = null
    orderingMode.value = 'rtl'
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
  pdfDocument = null
  orderingMode.value = 'rtl'
  if (fileInput.value) {
    fileInput.value.value = ''
  }
}

// Rotation control
const setRotation = (degrees) => {
  rotation.value = degrees
  // Redraw preview to update grid overlay
  if (sourceCanvas.value) {
    drawPreview(sourceCanvas.value)
  }
}

// Ordering mode control
const setOrderingMode = (mode) => {
  orderingMode.value = mode
  // Redraw preview to update grid overlay numbers
  if (sourceCanvas.value) {
    drawPreview(sourceCanvas.value)
  }
}

// Page position control
const setPagePosition = (pageNum) => {
  selectedPage.value = pageNum

  // Extract values before async operations
  const storedPdf = pdfDocument
  const pageCount = pdfPageCount.value

  // If this is a 2-page PDF, reload the preview with the selected page
  if (storedPdf && pageCount === 2) {
    storedPdf.getPage(pageNum).then(page => {
      const canvas = document.createElement('canvas')
      const context = canvas.getContext('2d')

      const viewport = page.getViewport({ scale: 4.2 })
      canvas.width = viewport.width
      canvas.height = viewport.height

      return page.render({
        canvasContext: context,
        viewport: viewport
      }).promise.then(() => canvas)
    }).then(canvas => {
      sourceCanvas.value = canvas
      drawPreview(canvas)
    })
  } else if (sourceCanvas.value) {
    // For single page images/PDFs, redraw preview to update numbers
    drawPreview(sourceCanvas.value)
  }
}

// Load preview
const loadPreview = (file) => {
  const isPdf = file.type === 'application/pdf'

  if (isPdf) {
    loadPdfPreview(file).then(result => {
      // Now we're in a .then() callback, setting refs directly
      if (result.error) {
        statusMessage.value = result.message
        statusType.value = 'error'
        if (result.clearFile) {
          selectedFile.value = null
          previewUrl.value = ''
          pdfPageCount.value = 0
          if (fileInput.value) {
            fileInput.value.value = ''
          }
        }
      } else {
        pdfPageCount.value = result.numPages
        pdfDocument = result.pdf
        sourceCanvas.value = result.canvas
        drawPreview(result.canvas)
      }
    }).catch(error => {
      console.error('Error loading preview:', error)
      statusMessage.value = `Error loading preview: ${error.message}`
      statusType.value = 'error'
    })
  } else {
    loadImagePreview(file).catch(error => {
      console.error('Error loading preview:', error)
      statusMessage.value = `Error loading preview: ${error.message}`
      statusType.value = 'error'
    })
  }
}

const loadImagePreview = (file) => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()

    reader.onload = (e) => {
      const img = new Image()
      img.onload = () => {
        // Create source canvas
        const canvas = document.createElement('canvas')
        canvas.width = img.width
        canvas.height = img.height
        const ctx = canvas.getContext('2d')
        ctx.drawImage(img, 0, 0)
        sourceCanvas.value = canvas

        // Draw preview
        drawPreview(canvas)
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
  console.log('loadPdfPreview: start')
  // Extract ALL ref values at the very beginning, before any awaits
  const currentSelectedPage = selectedPage.value
  const currentFileInput = fileInput.value

  console.log('loadPdfPreview: before arrayBuffer')
  const arrayBuffer = await file.arrayBuffer()
  console.log('loadPdfPreview: after arrayBuffer, before getDocument')
  const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise
  console.log('loadPdfPreview: after getDocument')

  // Check page count - AVOID accessing .value after await
  const numPages = pdf.numPages
  console.log('loadPdfPreview: setting pdfPageCount')

  // Store values to set, then set them all at once by returning them
  // and letting the caller set them
  if (numPages > 2) {
    // Return error state
    return {
      error: true,
      message: `⚠️ This PDF has ${numPages} pages. Only PDFs with 1 or 2 pages are supported. Please upload a different file.`,
      clearFile: true
    }
  }

  console.log('loadPdfPreview: before getPage')
  // Load the selected page (default is page 1)
  const page = await pdf.getPage(currentSelectedPage)
  console.log('loadPdfPreview: after getPage')

  const canvas = document.createElement('canvas')
  const context = canvas.getContext('2d')

  // Render at high resolution
  const viewport = page.getViewport({ scale: 4.2 })
  canvas.width = viewport.width
  canvas.height = viewport.height

  console.log('loadPdfPreview: before render')
  await page.render({
    canvasContext: context,
    viewport: viewport
  }).promise
  console.log('loadPdfPreview: after render')

  // Return the canvas and pdf to be set by caller
  return {
    error: false,
    canvas: canvas,
    pdf: pdf,
    numPages: numPages
  }
}

const drawPreview = (canvas) => {
  console.log('drawPreview: start')
  // Set previewUrl first so the canvas element gets rendered
  previewUrl.value = 'loaded'

  console.log('drawPreview: before nextTick')
  // Wait for Vue to render the canvas element
  nextTick().then(() => {
    console.log('drawPreview: inside nextTick callback')
    // Now safely access the ref
    const previewCanvasEl = previewCanvas.value

    if (!previewCanvasEl) {
      console.error('Preview canvas ref not available')
      return
    }

    // Apply rotation to the canvas before displaying
    const currentRotation = rotation.value
    const rotatedCanvas = applyRotation(canvas, currentRotation)

    const maxWidth = 600
    const maxHeight = 800
    let width = rotatedCanvas.width
    let height = rotatedCanvas.height

    // Scale down if needed
    const scale = Math.min(maxWidth / width, maxHeight / height, 1)
    width = width * scale
    height = height * scale

    previewCanvasEl.width = width
    previewCanvasEl.height = height

    const ctx = previewCanvasEl.getContext('2d')
    ctx.drawImage(rotatedCanvas, 0, 0, width, height)

    // Draw grid lines and numbers overlay
    // Use the rotated canvas dimensions for the overlay
    drawGridOverlay(ctx, width, height, rotatedCanvas.width, rotatedCanvas.height)

    console.log('drawPreview: done')
  })
}

const drawGridOverlay = (ctx, displayWidth, displayHeight, originalWidth, originalHeight) => {
  // The canvas is already rotated, so determine grid based on current dimensions
  const isPortrait = originalHeight > originalWidth

  // Get current values
  const currentOrderingMode = orderingMode.value
  const isFirstPage = selectedPage.value === 1

  // Calculate grid based on current orientation (after rotation)
  let cols, rows
  if (isPortrait) {
    cols = 2
    rows = 4
  } else {
    cols = 4
    rows = 2
  }

  const cellWidth = displayWidth / cols
  const cellHeight = displayHeight / rows

  // Draw grid lines
  ctx.strokeStyle = 'rgba(102, 126, 234, 0.8)'
  ctx.lineWidth = 2

  // Vertical lines
  for (let col = 1; col < cols; col++) {
    ctx.beginPath()
    ctx.moveTo(col * cellWidth, 0)
    ctx.lineTo(col * cellWidth, displayHeight)
    ctx.stroke()
  }

  // Horizontal lines
  for (let row = 1; row < rows; row++) {
    ctx.beginPath()
    ctx.moveTo(0, row * cellHeight)
    ctx.lineTo(displayWidth, row * cellHeight)
    ctx.stroke()
  }

  // Draw numbers in each cell
  ctx.font = 'bold 32px sans-serif'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'

  for (let row = 0; row < rows; row++) {
    for (let col = 0; col < cols; col++) {
      const position = row * cols + col
      // Use isPortrait since canvas is already rotated
      const pageNumber = getPageNumber(position, isPortrait, isFirstPage, currentOrderingMode)

      const x = col * cellWidth + cellWidth / 2
      const y = row * cellHeight + cellHeight / 2

      // Draw background circle
      ctx.fillStyle = 'rgba(102, 126, 234, 0.9)'
      ctx.beginPath()
      ctx.arc(x, y, 28, 0, Math.PI * 2)
      ctx.fill()

      // Draw number
      ctx.fillStyle = 'white'
      ctx.fillText(pageNumber.toString(), x, y)
    }
  }
}

// Apply rotation to canvas
const applyRotation = (sourceCanvas, rotationDegrees) => {
  if (rotationDegrees === 0) {
    return sourceCanvas
  }

  const canvas = document.createElement('canvas')
  const ctx = canvas.getContext('2d')

  // For 90 and 270 degrees, swap width and height
  if (rotationDegrees === 90 || rotationDegrees === 270) {
    canvas.width = sourceCanvas.height
    canvas.height = sourceCanvas.width
  } else {
    canvas.width = sourceCanvas.width
    canvas.height = sourceCanvas.height
  }

  // Move to center, rotate, then draw
  ctx.translate(canvas.width / 2, canvas.height / 2)
  ctx.rotate((rotationDegrees * Math.PI) / 180)
  ctx.drawImage(sourceCanvas, -sourceCanvas.width / 2, -sourceCanvas.height / 2)

  return canvas
}

// Image processing
const processDocument = () => {
  console.log('processDocument: START')
  if (!selectedFile.value || !sourceCanvas.value) return

  console.log('processDocument: setting initial state')
  processing.value = true
  progress.value = 0
  processedImages.value = []
  statusMessage.value = 'Processing document...'
  statusType.value = 'info'

  console.log('processDocument: extracting file')
  const file = selectedFile.value
  const isPdf = file.type === 'application/pdf'

  console.log('processDocument: setting progress to 20')
  progress.value = 20

  const processPromise = isPdf ? processPdf(file) : processImage()

  processPromise.then(pieces => {
    console.log('processDocument: got', pieces.length, 'pieces')
    console.log('processDocument: setting progress to 90')
    progress.value = 90
    console.log('processDocument: setting processedImages')
    processedImages.value = pieces
    console.log('processDocument: setting progress to 100')
    progress.value = 100
    console.log('processDocument: setting success message')
    statusMessage.value = `Successfully generated ${pieces.length} A7 images!`
    statusType.value = 'success'
    console.log('processDocument: SUCCESS')
  }).catch(error => {
    console.error('processDocument: ERROR', error)
    statusMessage.value = `Error: ${error.message}`
    statusType.value = 'error'
  }).finally(() => {
    console.log('processDocument: FINALLY')
    processing.value = false
  })
}

const processPdf = (file) => {
  console.log('processPdf: START')
  // Extract all values at the beginning
  const currentSelectedPage = selectedPage.value
  const currentRotation = rotation.value
  const currentOrderingMode = orderingMode.value
  const storedPdf = pdfDocument
  const baseName = file.name.replace(/\.\w+$/, '')

  // Use the already loaded PDF document if available
  const pdfPromise = storedPdf
    ? Promise.resolve(storedPdf)
    : file.arrayBuffer().then(arrayBuffer => {
        console.log('processPdf: after arrayBuffer')
        return pdfjsLib.getDocument({ data: arrayBuffer }).promise
      })

  return pdfPromise.then(pdf => {
    console.log('processPdf: got pdf, getting page')
    return pdf.getPage(currentSelectedPage)
  }).then(page => {
    console.log('processPdf: got page, creating canvas')
    const canvas = document.createElement('canvas')
    const context = canvas.getContext('2d')

    // Render at high resolution
    const viewport = page.getViewport({ scale: 4.2 })
    canvas.width = viewport.width
    canvas.height = viewport.height

    console.log('processPdf: rendering page')
    return page.render({
      canvasContext: context,
      viewport: viewport
    }).promise.then(() => canvas)
  }).then(canvas => {
    console.log('processPdf: rendered, applying rotation')
    // Apply rotation
    const rotatedCanvas = applyRotation(canvas, currentRotation)

    // Determine if this is the first page
    const isFirstPage = currentSelectedPage === 1
    const startIndex = isFirstPage ? 1 : 9

    console.log('processPdf: splitting into A7')
    // Split this page into A7 pieces
    return splitIntoA7(rotatedCanvas, baseName, startIndex, isFirstPage, currentOrderingMode)
  }).then(pieces => {
    console.log('processPdf: done, returning pieces')
    return pieces
  })
}

const processImage = () => {
  console.log('processImage: START')
  // Extract all ref values at the beginning
  const currentSelectedPage = selectedPage.value
  const currentRotation = rotation.value
  const currentOrderingMode = orderingMode.value
  const currentSourceCanvas = sourceCanvas.value
  const fileName = selectedFile.value.name

  // Use the already loaded source canvas
  const baseName = fileName.replace(/\.\w+$/, '')

  // Apply rotation
  const rotatedCanvas = applyRotation(currentSourceCanvas, currentRotation)

  // Determine if this is the first page
  const isFirstPage = currentSelectedPage === 1
  const startIndex = isFirstPage ? 1 : 9

  console.log('processImage: calling splitIntoA7')
  return splitIntoA7(rotatedCanvas, baseName, startIndex, isFirstPage, currentOrderingMode).then(pieces => {
    console.log('processImage: done, returning pieces')
    return pieces
  })
}

// Get page number based on ordering mode
const getPageNumber = (position, isPortrait, isFirstPage, mode) => {
  // For standard mode, just use sequential numbering
  if (mode === 'standard') {
    return isFirstPage ? position + 1 : position + 9
  }

  // RTL Comix Booklet Template
  if (mode === 'rtl') {
    if (isPortrait) {
      // Portrait: 2 cols × 4 rows
      // First page: top row [7,3], 2nd row [10,14], 3rd row [5,1], last row [12,16]
      // Second page: top row [11,15], 2nd row [6,2], 3rd row [9,13], last row [8,4]
      const rtlPortraitFirst = [7, 3, 10, 14, 5, 1, 12, 16]
      const rtlPortraitSecond = [11, 15, 6, 2, 9, 13, 8, 4]
      return isFirstPage ? rtlPortraitFirst[position] : rtlPortraitSecond[position]
    } else {
      // Landscape: 4 cols × 2 rows
      const rtlLandscapeFirst = [3, 14, 1, 16, 7, 10, 5, 12]
      const rtlLandscapeSecond = [15, 2, 13, 4, 11, 6, 9, 8]
      return isFirstPage ? rtlLandscapeFirst[position] : rtlLandscapeSecond[position]
    }
  }

  // LTR Comix Booklet Template
  if (mode === 'ltr') {
    if (isPortrait) {
      // Portrait: 2 cols × 4 rows
      // LTR portrait mapping (TBD - using standard for now)
      const ltrPortraitFirst = [1, 2, 3, 4, 5, 6, 7, 8]
      const ltrPortraitSecond = [9, 10, 11, 12, 13, 14, 15, 16]
      return isFirstPage ? ltrPortraitFirst[position] : ltrPortraitSecond[position]
    } else {
      // Landscape: 4 cols × 2 rows
      // LTR landscape mapping (mirror of RTL - TBD)
      const ltrLandscapeFirst = [1, 2, 3, 4, 5, 6, 7, 8]
      const ltrLandscapeSecond = [9, 10, 11, 12, 13, 14, 15, 16]
      return isFirstPage ? ltrLandscapeFirst[position] : ltrLandscapeSecond[position]
    }
  }

  // Fallback to standard
  return isFirstPage ? position + 1 : position + 9
}

const splitIntoA7 = async (canvas, baseName, startIndex, isFirstPage, currentOrderingMode) => {
  console.log('splitIntoA7: START')
  const pieces = []
  const width = canvas.width
  const height = canvas.height

  console.log('splitIntoA7: canvas size', width, 'x', height)
  // Detect orientation: portrait if height > width, landscape otherwise
  const isPortrait = height > width
  console.log('splitIntoA7: isPortrait=', isPortrait)

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

  // Note: Status message is set by the calling function

  console.log('splitIntoA7: starting loop, rows=', rows, 'cols=', cols)
  for (let row = 0; row < rows; row++) {
    for (let col = 0; col < cols; col++) {
      console.log('splitIntoA7: processing piece', row, col)
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

      console.log('splitIntoA7: before toBlob/toDataURL')
      // Convert to JPG with high quality
      const dataUrl = pieceCanvas.toBlob ?
        await new Promise(resolve => {
          pieceCanvas.toBlob(blob => {
            resolve(URL.createObjectURL(blob))
          }, 'image/jpeg', 0.95)
        }) :
        pieceCanvas.toDataURL('image/jpeg', 0.95)

      console.log('splitIntoA7: after toBlob/toDataURL')
      // Calculate position in the grid (0-7)
      const position = row * cols + col
      // Get the page number based on ordering mode
      const pageNumber = getPageNumber(position, isPortrait, isFirstPage, currentOrderingMode)

      console.log('splitIntoA7: pushing piece', pageNumber)
      pieces.push({
        filename: `${baseName}_${pageNumber}.jpg`,
        dataUrl: dataUrl,
        canvas: pieceCanvas
      })
    }
  }

  console.log('splitIntoA7: done, returning', pieces.length, 'pieces')
  return pieces
}

// Download functionality - create and download as ZIP
const downloadAllAsZip = async () => {
  try {
    statusMessage.value = 'Creating ZIP file...'
    statusType.value = 'info'

    const zip = new JSZip()

    // Add each image to the zip file
    for (const img of processedImages.value) {
      // Convert dataUrl/blob to actual blob
      let blob
      if (img.dataUrl.startsWith('blob:')) {
        const response = await fetch(img.dataUrl)
        blob = await response.blob()
      } else {
        const response = await fetch(img.dataUrl)
        blob = await response.blob()
      }

      zip.file(img.filename, blob)
    }

    // Generate zip file
    const zipBlob = await zip.generateAsync({ type: 'blob' })

    // Create download link for the zip file
    const url = URL.createObjectURL(zipBlob)
    const link = document.createElement('a')
    link.href = url

    // Use original filename (without extension) + .zip
    const originalName = selectedFile.value.name.replace(/\.\w+$/, '')
    link.download = `${originalName}_a7_images.zip`

    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)

    // Clean up
    setTimeout(() => URL.revokeObjectURL(url), 100)

    statusMessage.value = 'ZIP file downloaded successfully!'
    statusType.value = 'success'
  } catch (error) {
    console.error('Error creating ZIP:', error)
    statusMessage.value = `Error creating ZIP: ${error.message}`
    statusType.value = 'error'
  }
}
</script>
