<template>
  <div class="container">
    <div class="app-nav">
      <span class="nav-current">📄 A7 Splitter</span>
      <span class="nav-separator">|</span>
      <a href="pager.html" class="nav-link">📖 Pager</a>
    </div>
    <h1>📄 A4 to A7 Document Splitter</h1>
    <p class="subtitle">Upload 1 or 2 pages (portrait or landscape), then split each into 8 properly-sized JPG images</p>

    <!-- Upload Area -->
    <div
      v-if="selectedFiles.length === 0"
      class="upload-area"
      :class="{ dragging: isDragging }"
      @click="triggerFileInput"
      @dragover.prevent="isDragging = true"
      @dragleave.prevent="isDragging = false"
      @drop.prevent="handleDrop"
    >
      <div class="upload-icon">📎</div>
      <div class="upload-text">Click to upload or drag and drop</div>
      <div class="upload-hint">Upload 1 or 2 pages (JPG, PNG, PDF)</div>
      <input
        ref="fileInput"
        type="file"
        class="file-input"
        accept="image/*,application/pdf"
        multiple
        @change="handleFileSelect"
      >
    </div>

    <!-- File Info & Preview -->
    <div v-if="selectedFiles.length > 0 && !processedImages.length" class="file-info">
      <div class="file-cards-row">
        <!-- Selected Files Card -->
        <div class="info-card files-card">
          <div class="card-header">
            <div class="card-title">
              <span class="card-emoji">📁</span>
              <h3>Selected File{{ selectedFiles.length > 1 ? 's' : '' }}</h3>
            </div>
            <button
              class="btn btn-secondary btn-small"
              @click="reset"
              :disabled="processing"
            >
              Choose Different File{{ selectedFiles.length > 1 ? 's' : '' }}
            </button>
          </div>
          <div v-if="selectedFiles.length === 1 && pdfPageCount !== 2" class="file-name">📄 {{ selectedFiles[0].name }}</div>
          <div v-else-if="selectedFiles.length === 1 && pdfPageCount === 2" class="file-name">📄 {{ selectedFiles[0].name }} <span class="file-badge">2 pages</span></div>
          <div v-else class="file-list">
            <div v-for="(file, idx) in selectedFiles" :key="idx" class="file-item">
              📄 {{ idx + 1 }}. {{ file.name }}
            </div>
          </div>
        </div>

        <!-- Output Name Card -->
        <div class="info-card output-card">
          <div class="card-header">
            <div class="card-title">
              <span class="card-emoji">✏️</span>
              <h3>Output Name</h3>
            </div>
          </div>
          <input
            type="text"
            v-model="outputBaseName"
            :disabled="processing"
            placeholder="Enter base name for output files"
            class="file-name-input-card"
          >
          <div class="card-hint">💡 Files will be named: [name]_1.jpg, [name]_2.jpg, etc.</div>
        </div>
      </div>

      <!-- Dual File Preview (Two Files Mode or Two-Page PDF) -->
      <div v-if="previewUrl === 'dual-files' || (previewUrl && pdfPageCount === 2)" class="dual-preview-section">
        <div class="preview-header">
          <h3>Preview:</h3>
        </div>
        <div class="info-banner">
          <span v-if="selectedFiles.length === 2">
            ℹ️ Two files detected! Clicking "Generate" will create all 16 images (8 from each file) and automatically download them as a ZIP file.
          </span>
          <span v-else-if="pdfPageCount === 2">
            ℹ️ Two-page PDF detected! Clicking "Generate" will create all 16 images (8 from each page) and automatically download them as a ZIP file.
          </span>
        </div>
        <div class="dual-preview-container">
          <!-- Page 1 Preview (left/top) -->
          <div class="preview-item">
            <div class="preview-label">
              <span v-if="selectedFiles.length === 2">Page 1: {{ selectedFiles[page1ImageIndex]?.name }}</span>
              <span v-else-if="pdfPageCount === 2">Page 1 (from PDF Page {{ pdfPage1Index }})</span>
              <span v-else>Page 1</span>
            </div>
            <canvas
              ref="previewCanvasPage1"
              class="source-preview-canvas"
            ></canvas>
          </div>
          <!-- Page 2 Preview (right/bottom) -->
          <div class="preview-item">
            <div class="preview-label">
              <span v-if="selectedFiles.length === 2">Page 2: {{ selectedFiles[page1ImageIndex === 0 ? 1 : 0]?.name }}</span>
              <span v-else-if="pdfPageCount === 2">Page 2 (from PDF Page {{ pdfPage1Index === 1 ? 2 : 1 }})</span>
              <span v-else>Page 2</span>
            </div>
            <canvas
              ref="previewCanvasPage2"
              class="source-preview-canvas"
            ></canvas>
          </div>
        </div>

        <!-- Modern Settings Controls -->
        <div class="settings-panel">
          <!-- Page Order Selector (Two Files Mode or Two-Page PDF) -->
          <div v-if="selectedFiles.length === 2 || pdfPageCount === 2" class="setting-card">
            <div class="setting-header">
              <span class="setting-icon">📄</span>
              <span class="setting-title">Page Order</span>
            </div>
            <!-- Two separate files -->
            <div v-if="selectedFiles.length === 2" class="page-order-selector">
              <div class="page-order-item">
                <span class="page-label">Page 1:</span>
                <select v-model="page1ImageIndex" :disabled="processing" class="page-select" @change="drawDualPreviews">
                  <option :value="0">{{ selectedFiles[0]?.name }}</option>
                  <option :value="1">{{ selectedFiles[1]?.name }}</option>
                </select>
              </div>
              <div class="page-order-item">
                <span class="page-label">Page 2:</span>
                <span class="page-value">{{ selectedFiles[page1ImageIndex === 0 ? 1 : 0]?.name }}</span>
              </div>
            </div>
            <!-- Two-page PDF -->
            <div v-else-if="pdfPageCount === 2" class="page-order-selector">
              <div class="page-order-item">
                <span class="page-label">Page 1:</span>
                <select v-model="pdfPage1Index" :disabled="processing" class="page-select" @change="drawDualPreviews">
                  <option :value="1">PDF Page 1</option>
                  <option :value="2">PDF Page 2</option>
                </select>
              </div>
              <div class="page-order-item">
                <span class="page-label">Page 2:</span>
                <span class="page-value">PDF Page {{ pdfPage1Index === 1 ? 2 : 1 }}</span>
              </div>
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

      <!-- Source Preview (Single File Mode Only) -->
      <div v-if="previewUrl && previewUrl !== 'dual-files' && previewUrl !== 'multiple-files'" class="source-preview-section">
        <div class="preview-header">
          <h3>Preview:</h3>
          <div v-if="pdfPageCount > 1" class="page-info">
            Viewing: Page {{ selectedPage }} of {{ pdfPageCount }}
          </div>
        </div>
        <div v-if="pdfPageCount === 2" class="info-banner">
          ℹ️ Two-page PDF detected! Clicking "Generate" will create all 16 images (8 from each page) and automatically download them as a ZIP file.
        </div>
        <div class="preview-container">
          <canvas
            ref="previewCanvas"
            class="source-preview-canvas"
          ></canvas>
        </div>

        <!-- Modern Settings Controls -->
        <div class="settings-panel">
          <!-- Page Position Switch (Single File Mode) -->
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
          :disabled="processing || (selectedFiles.length === 1 && !previewUrl) || selectedFiles.length === 0"
        >
          {{ processing ? 'Processing...' :
             (selectedFiles.length === 2 ? 'Generate All 16 Images & Download' :
              (pdfPageCount === 2 ? 'Generate All 16 Images & Download' : 'Split into Images')) }}
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
const selectedFiles = ref([]) // Array to hold 1 or 2 files
const fileInput = ref(null)
const previewCanvas = ref(null)
const previewCanvas2 = ref(null) // Second canvas for dual-file mode
const previewCanvasPage1 = ref(null) // Canvas for page 1 in dual mode (always on top)
const previewCanvasPage2 = ref(null) // Canvas for page 2 in dual mode (always on bottom)
const isDragging = ref(false)
const processing = ref(false)
const progress = ref(0)
const processedImages = ref([])
const statusMessage = ref('')
const statusType = ref('info')
const previewUrl = ref('')
const rotation = ref(0)
const sourceCanvas = ref(null)
const sourceCanvas2 = ref(null) // Second source canvas for dual-file mode
const selectedPage = ref(1) // 1 for first page, 2 for second page (used in single-file mode)
const pdfPageCount = ref(0)
// Store PDF document in plain variable to avoid Vue reactivity breaking PDF.js private fields
let pdfDocument = null
const orderingMode = ref('rtl') // 'standard', 'rtl', 'ltr'
const pageSize = ref('a7') // 'a7', 'a6', 'a5', 'a4'
const page1ImageIndex = ref(0) // Which file is page 1 when 2 files are uploaded (0 or 1)
const pdfPage1Index = ref(1) // Which PDF page is page 1 when a 2-page PDF is uploaded (1 or 2)
const outputBaseName = ref('') // Custom base name for output files

// Page size dimensions in mm (width x height for portrait orientation)
const pageSizes = {
  a7: { width: 74, height: 105 },
  a6: { width: 105, height: 148 },
  a5: { width: 148, height: 210 },
  a4: { width: 210, height: 297 }
}

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
  const files = Array.from(event.target.files).slice(0, 2) // Limit to 2 files
  if (files.length > 0) {
    selectedFiles.value = files
    processedImages.value = []
    statusMessage.value = ''
    rotation.value = 0
    selectedPage.value = 1
    pdfPageCount.value = 0
    pdfDocument = null
    orderingMode.value = 'rtl'
    pageSize.value = 'a7'
    page1ImageIndex.value = 0
    pdfPage1Index.value = 1
    outputBaseName.value = files[0].name.replace(/\.\w+$/, '')

    // Load preview
    if (files.length === 1) {
      loadPreview(files[0])
    } else if (files.length === 2) {
      // For 2 files, load both previews
      loadDualPreviews(files[0], files[1])
    }
  }
}

const handleDrop = (event) => {
  isDragging.value = false
  const files = Array.from(event.dataTransfer.files).slice(0, 2) // Limit to 2 files
  if (files.length > 0) {
    selectedFiles.value = files
    processedImages.value = []
    statusMessage.value = ''
    rotation.value = 0
    selectedPage.value = 1
    pdfPageCount.value = 0
    pdfDocument = null
    pageSize.value = 'a7'
    orderingMode.value = 'rtl'
    page1ImageIndex.value = 0
    pdfPage1Index.value = 1
    outputBaseName.value = files[0].name.replace(/\.\w+$/, '')

    // Load preview
    if (files.length === 1) {
      loadPreview(files[0])
    } else if (files.length === 2) {
      // For 2 files, load both previews
      loadDualPreviews(files[0], files[1])
    }
  }
}

const reset = () => {
  selectedFiles.value = []
  processedImages.value = []
  statusMessage.value = ''
  progress.value = 0
  previewUrl.value = ''
  rotation.value = 0
  sourceCanvas.value = null
  sourceCanvas2.value = null
  selectedPage.value = 1
  pdfPageCount.value = 0
  pdfDocument = null
  orderingMode.value = 'rtl'
  pageSize.value = 'a7'
  page1ImageIndex.value = 0
  pdfPage1Index.value = 1
  outputBaseName.value = ''
  if (fileInput.value) {
    fileInput.value.value = ''
  }
}

// Rotation control
const setRotation = (degrees) => {
  rotation.value = degrees
  // Redraw preview to update grid overlay
  const isTwoPagePdf = pdfPageCount.value === 2
  if ((selectedFiles.value.length === 2 || isTwoPagePdf) && sourceCanvas.value && sourceCanvas2.value) {
    // Dual-file mode or 2-page PDF: redraw both previews
    drawDualPreviews()
  } else if (sourceCanvas.value) {
    // Single-file mode: redraw single preview
    drawPreview(sourceCanvas.value)
  }
}

// Ordering mode control
const setOrderingMode = (mode) => {
  orderingMode.value = mode
  // Redraw preview to update grid overlay numbers
  const isTwoPagePdf = pdfPageCount.value === 2
  if ((selectedFiles.value.length === 2 || isTwoPagePdf) && sourceCanvas.value && sourceCanvas2.value) {
    // Dual-file mode or 2-page PDF: redraw both previews
    drawDualPreviews()
  } else if (sourceCanvas.value) {
    // Single-file mode: redraw single preview
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

        // Handle 2-page PDF like dual files
        if (result.numPages === 2) {
          sourceCanvas.value = result.canvas1
          sourceCanvas2.value = result.canvas2
          drawDualPreviews()
        } else {
          sourceCanvas.value = result.canvas
          drawPreview(result.canvas)
        }
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

  // For 2-page PDFs, load both pages
  if (numPages === 2) {
    console.log('loadPdfPreview: loading both pages')
    const page1 = await pdf.getPage(1)
    const page2 = await pdf.getPage(2)

    const canvas1 = document.createElement('canvas')
    const context1 = canvas1.getContext('2d')
    const viewport1 = page1.getViewport({ scale: 4.2 })
    canvas1.width = viewport1.width
    canvas1.height = viewport1.height
    await page1.render({ canvasContext: context1, viewport: viewport1 }).promise

    const canvas2 = document.createElement('canvas')
    const context2 = canvas2.getContext('2d')
    const viewport2 = page2.getViewport({ scale: 4.2 })
    canvas2.width = viewport2.width
    canvas2.height = viewport2.height
    await page2.render({ canvasContext: context2, viewport: viewport2 }).promise

    return {
      error: false,
      canvas1: canvas1,
      canvas2: canvas2,
      pdf: pdf,
      numPages: numPages
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

// Load and display previews for both files in dual-file mode
const loadDualPreviews = async (file1, file2) => {
  try {
    // Load both files
    const canvas1 = await loadFileToCanvas(file1)
    const canvas2 = await loadFileToCanvas(file2)

    sourceCanvas.value = canvas1
    sourceCanvas2.value = canvas2

    // Draw both previews
    drawDualPreviews()
  } catch (error) {
    console.error('Error loading dual previews:', error)
    statusMessage.value = `Error loading previews: ${error.message}`
    statusType.value = 'error'
  }
}

// Load a single file (image or PDF) to canvas
const loadFileToCanvas = (file) => {
  return new Promise((resolve, reject) => {
    const isPdf = file.type === 'application/pdf'

    if (isPdf) {
      // Load PDF
      file.arrayBuffer().then(arrayBuffer => {
        return pdfjsLib.getDocument({ data: arrayBuffer }).promise
      }).then(pdf => {
        return pdf.getPage(1) // Always load first page
      }).then(page => {
        const canvas = document.createElement('canvas')
        const context = canvas.getContext('2d')

        const viewport = page.getViewport({ scale: 4.2 })
        canvas.width = viewport.width
        canvas.height = viewport.height

        return page.render({
          canvasContext: context,
          viewport: viewport
        }).promise.then(() => canvas)
      }).then(resolve).catch(reject)
    } else {
      // Load image
      const reader = new FileReader()
      reader.onload = (e) => {
        const img = new Image()
        img.onload = () => {
          const canvas = document.createElement('canvas')
          canvas.width = img.width
          canvas.height = img.height
          const ctx = canvas.getContext('2d')
          ctx.drawImage(img, 0, 0)
          resolve(canvas)
        }
        img.onerror = () => reject(new Error(`Failed to load image: ${file.name}`))
        img.src = e.target.result
      }
      reader.onerror = () => reject(new Error(`Failed to read file: ${file.name}`))
      reader.readAsDataURL(file)
    }
  })
}

// Draw both previews with grid overlays
const drawDualPreviews = () => {
  console.log('drawDualPreviews: start')
  previewUrl.value = 'dual-files'

  nextTick().then(() => {
    const previewCanvasElPage1 = previewCanvasPage1.value
    const previewCanvasElPage2 = previewCanvasPage2.value

    if (!previewCanvasElPage1 || !previewCanvasElPage2 || !sourceCanvas.value || !sourceCanvas2.value) {
      console.error('Preview canvases not available')
      return
    }

    const currentRotation = rotation.value
    const currentOrderingMode = orderingMode.value
    const currentPage1Index = page1ImageIndex.value
    const currentPdfPage1Index = pdfPage1Index.value
    const isTwoPagePdf = pdfPageCount.value === 2

    // Determine which source canvas is page 1 and which is page 2
    let page1SourceCanvas, page2SourceCanvas

    if (isTwoPagePdf) {
      // For 2-page PDFs, use pdfPage1Index to determine order
      page1SourceCanvas = currentPdfPage1Index === 1 ? sourceCanvas.value : sourceCanvas2.value
      page2SourceCanvas = currentPdfPage1Index === 1 ? sourceCanvas2.value : sourceCanvas.value
    } else {
      // For dual separate files, use page1ImageIndex
      page1SourceCanvas = currentPage1Index === 0 ? sourceCanvas.value : sourceCanvas2.value
      page2SourceCanvas = currentPage1Index === 0 ? sourceCanvas2.value : sourceCanvas.value
    }

    // Apply rotation and draw page 1 to the top preview canvas
    const rotatedCanvas1 = applyRotation(page1SourceCanvas, currentRotation)
    drawSinglePreviewToCanvas(rotatedCanvas1, previewCanvasElPage1, true, currentOrderingMode)

    // Apply rotation and draw page 2 to the bottom preview canvas
    const rotatedCanvas2 = applyRotation(page2SourceCanvas, currentRotation)
    drawSinglePreviewToCanvas(rotatedCanvas2, previewCanvasElPage2, false, currentOrderingMode)

    console.log('drawDualPreviews: done')
  })
}

// Draw a single preview to a specific canvas element
const drawSinglePreviewToCanvas = (sourceCanvas, targetCanvasEl, isFirstPage, orderingMode) => {
  const maxWidth = 500
  const maxHeight = 700
  let width = sourceCanvas.width
  let height = sourceCanvas.height

  // Scale down if needed
  const scale = Math.min(maxWidth / width, maxHeight / height, 1)
  width = width * scale
  height = height * scale

  targetCanvasEl.width = width
  targetCanvasEl.height = height

  const ctx = targetCanvasEl.getContext('2d')
  ctx.drawImage(sourceCanvas, 0, 0, width, height)

  // Draw grid lines and numbers overlay
  drawGridOverlay(ctx, width, height, sourceCanvas.width, sourceCanvas.height, isFirstPage, orderingMode)
}

// Update drawGridOverlay signature to accept isFirstPage and orderingMode
const drawGridOverlay = (ctx, displayWidth, displayHeight, originalWidth, originalHeight, isFirstPage = true, mode = 'rtl') => {
  // The canvas is already rotated, so determine grid based on current dimensions
  const isPortrait = originalHeight > originalWidth

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
      const pageNumber = getPageNumber(position, isPortrait, isFirstPage, mode)

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
    const currentOrderingMode = orderingMode.value
    const isFirstPage = selectedPage.value === 1
    drawGridOverlay(ctx, width, height, rotatedCanvas.width, rotatedCanvas.height, isFirstPage, currentOrderingMode)

    console.log('drawPreview: done')
  })
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
  if (selectedFiles.value.length === 0) return

  // Check if in dual-file mode (2 files uploaded)
  const isDualFileMode = selectedFiles.value.length === 2

  // In single file mode, require preview to be loaded
  if (!isDualFileMode && !sourceCanvas.value) return

  console.log('processDocument: setting initial state')
  processing.value = true
  progress.value = 0
  processedImages.value = []
  statusMessage.value = 'Processing document...'
  statusType.value = 'info'

  let processPromise

  if (isDualFileMode) {
    // Process two separate files as page 1 and page 2
    console.log('processDocument: processing two files')
    statusMessage.value = 'Processing both files...'

    const currentPage1Index = page1ImageIndex.value
    const file1 = selectedFiles.value[currentPage1Index]
    const file2 = selectedFiles.value[currentPage1Index === 0 ? 1 : 0]

    progress.value = 20

    processPromise = Promise.all([
      processFileAsPage(file1, 1),
      processFileAsPage(file2, 2)
    ]).then(([page1Pieces, page2Pieces]) => {
      console.log('processDocument: combining pieces from both files')
      return [...page1Pieces, ...page2Pieces]
    })
  } else {
    // Single file mode - existing logic
    console.log('processDocument: extracting file')
    const file = selectedFiles.value[0]
    const isPdf = file.type === 'application/pdf'
    const isTwoPagePdf = isPdf && pdfPageCount.value === 2

    console.log('processDocument: setting progress to 20')
    progress.value = 20

    // For two-page PDFs, process both pages and combine results
    if (isTwoPagePdf) {
      console.log('processDocument: processing two-page PDF')
      statusMessage.value = 'Processing both pages...'

      // Respect the page order selection
      const currentPdfPage1Index = pdfPage1Index.value
      const pdfPageNum1 = currentPdfPage1Index // 1 or 2
      const pdfPageNum2 = currentPdfPage1Index === 1 ? 2 : 1 // the other page

      processPromise = Promise.all([
        processPdf(file, pdfPageNum1),
        processPdf(file, pdfPageNum2)
      ]).then(([page1Pieces, page2Pieces]) => {
        console.log('processDocument: combining pieces from both pages')
        return [...page1Pieces, ...page2Pieces]
      })
    } else {
      // For single-page PDFs and images, process normally
      processPromise = isPdf ? processPdf(file) : processImage()
    }
  }

  processPromise.then(pieces => {
    console.log('processDocument: got', pieces.length, 'pieces')
    console.log('processDocument: setting progress to 90')
    progress.value = 90
    console.log('processDocument: setting processedImages')
    processedImages.value = pieces
    console.log('processDocument: setting progress to 100')
    progress.value = 100
    console.log('processDocument: setting success message')
    statusMessage.value = `Successfully generated ${pieces.length} images!`
    statusType.value = 'success'
    console.log('processDocument: SUCCESS')

    // Auto-download for two-page PDFs or dual-file mode
    if (isDualFileMode || (selectedFiles.value[0]?.type === 'application/pdf' && pdfPageCount.value === 2)) {
      console.log('processDocument: auto-downloading')
      setTimeout(() => {
        downloadAllAsZip()
      }, 500)
    }
  }).catch(error => {
    console.error('processDocument: ERROR', error)
    statusMessage.value = `Error: ${error.message}`
    statusType.value = 'error'
  }).finally(() => {
    console.log('processDocument: FINALLY')
    processing.value = false
  })
}

// Process a single file as a specific page number
const processFileAsPage = (file, pageNumber) => {
  console.log('processFileAsPage: START, pageNumber=', pageNumber)
  const currentRotation = rotation.value
  const currentOrderingMode = orderingMode.value
  const currentPageSize = pageSize.value
  const baseName = outputBaseName.value || file.name.replace(/\.\w+$/, '')
  const isPdf = file.type === 'application/pdf'

  if (isPdf) {
    // Load and process PDF
    return file.arrayBuffer().then(arrayBuffer => {
      return pdfjsLib.getDocument({ data: arrayBuffer }).promise
    }).then(pdf => {
      return pdf.getPage(1) // Always use first page of each PDF
    }).then(page => {
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
      const rotatedCanvas = applyRotation(canvas, currentRotation)
      const isFirstPage = pageNumber === 1
      return splitIntoA7(rotatedCanvas, baseName, pageNumber, isFirstPage, currentOrderingMode, currentPageSize)
    })
  } else {
    // Load and process image
    return new Promise((resolve, reject) => {
      const reader = new FileReader()
      reader.onload = (e) => {
        const img = new Image()
        img.onload = () => {
          const canvas = document.createElement('canvas')
          canvas.width = img.width
          canvas.height = img.height
          const ctx = canvas.getContext('2d')
          ctx.drawImage(img, 0, 0)

          const rotatedCanvas = applyRotation(canvas, currentRotation)
          const isFirstPage = pageNumber === 1
          resolve(splitIntoA7(rotatedCanvas, baseName, pageNumber, isFirstPage, currentOrderingMode, currentPageSize))
        }
        img.onerror = () => reject(new Error(`Failed to load image: ${file.name}`))
        img.src = e.target.result
      }
      reader.onerror = () => reject(new Error(`Failed to read file: ${file.name}`))
      reader.readAsDataURL(file)
    })
  }
}

const processPdf = (file, pageNum = null) => {
  console.log('processPdf: START, pageNum=', pageNum)
  // Extract all values at the beginning
  const currentSelectedPage = pageNum !== null ? pageNum : selectedPage.value
  const currentRotation = rotation.value
  const currentOrderingMode = orderingMode.value
  const currentPageSize = pageSize.value
  const storedPdf = pdfDocument
  const baseName = outputBaseName.value || file.name.replace(/\.\w+$/, '')

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

    console.log('processPdf: splitting into pieces')
    // Split this page into pieces
    return splitIntoA7(rotatedCanvas, baseName, startIndex, isFirstPage, currentOrderingMode, currentPageSize)
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
  const currentPageSize = pageSize.value
  const currentSourceCanvas = sourceCanvas.value
  const fileName = selectedFiles.value[0].name

  // Use the already loaded source canvas
  const baseName = outputBaseName.value || fileName.replace(/\.\w+$/, '')

  // Apply rotation
  const rotatedCanvas = applyRotation(currentSourceCanvas, currentRotation)

  // Determine if this is the first page
  const isFirstPage = currentSelectedPage === 1
  const startIndex = isFirstPage ? 1 : 9

  console.log('processImage: calling splitIntoA7')
  return splitIntoA7(rotatedCanvas, baseName, startIndex, isFirstPage, currentOrderingMode, currentPageSize).then(pieces => {
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

// Convert mm to pixels at 300 DPI
const mmToPx = (mm) => {
  const dpi = 300
  return Math.round((mm / 25.4) * dpi)
}

const splitIntoA7 = async (canvas, baseName, startIndex, isFirstPage, currentOrderingMode, pageSizeKey) => {
  console.log('splitIntoA7: START')
  const pieces = []
  const width = canvas.width
  const height = canvas.height

  console.log('splitIntoA7: canvas size', width, 'x', height)
  // Detect orientation: portrait if height > width, landscape otherwise
  const isPortrait = height > width
  console.log('splitIntoA7: isPortrait=', isPortrait)

  // Determine grid layout based on orientation
  let cols, rows
  if (isPortrait) {
    cols = 2
    rows = 4
  } else {
    cols = 4
    rows = 2
  }

  // Calculate dimensions of each piece from source
  const pieceWidth = width / cols
  const pieceHeight = height / rows

  // Get target page size dimensions in mm and convert to pixels at 300 DPI
  const targetSize = pageSizes[pageSizeKey]
  const pageWidthPx = mmToPx(targetSize.width)
  const pageHeightPx = mmToPx(targetSize.height)

  // Fixed bleeds: 3mm on all sides
  const bleed = mmToPx(3)

  // Calculate available space for content (page size minus bleeds)
  const contentWidth = pageWidthPx - (bleed * 2)
  const contentHeight = pageHeightPx - (bleed * 2)

  console.log('splitIntoA7: starting loop, rows=', rows, 'cols=', cols)
  for (let row = 0; row < rows; row++) {
    for (let col = 0; col < cols; col++) {
      console.log('splitIntoA7: processing piece', row, col)

      // Extract piece from source canvas
      const pieceCanvas = document.createElement('canvas')
      pieceCanvas.width = pieceWidth
      pieceCanvas.height = pieceHeight
      const pieceCtx = pieceCanvas.getContext('2d')
      pieceCtx.imageSmoothingEnabled = true
      pieceCtx.imageSmoothingQuality = 'high'

      const srcX = col * pieceWidth
      const srcY = row * pieceHeight

      pieceCtx.drawImage(
        canvas,
        srcX, srcY, pieceWidth, pieceHeight,
        0, 0, pieceWidth, pieceHeight
      )

      // Create final canvas with page size
      const finalCanvas = document.createElement('canvas')
      finalCanvas.width = pageWidthPx
      finalCanvas.height = pageHeightPx
      const finalCtx = finalCanvas.getContext('2d')
      finalCtx.imageSmoothingEnabled = true
      finalCtx.imageSmoothingQuality = 'high'

      // Fill with white background
      finalCtx.fillStyle = 'white'
      finalCtx.fillRect(0, 0, finalCanvas.width, finalCanvas.height)

      // Calculate scale factor to fit piece within content area
      const scaleX = contentWidth / pieceWidth
      const scaleY = contentHeight / pieceHeight
      const scale = Math.min(scaleX, scaleY)

      // Calculate scaled dimensions
      const scaledWidth = pieceWidth * scale
      const scaledHeight = pieceHeight * scale

      // Center the scaled piece within the content area (with bleeds)
      const x = bleed + (contentWidth - scaledWidth) / 2
      const y = bleed + (contentHeight - scaledHeight) / 2

      // Draw the scaled piece
      finalCtx.drawImage(pieceCanvas, x, y, scaledWidth, scaledHeight)

      console.log('splitIntoA7: before toBlob')
      // Convert to blob
      const blob = await new Promise(resolve => {
        finalCanvas.toBlob(resolve, 'image/jpeg', 0.95)
      })
      const dataUrl = URL.createObjectURL(blob)

      console.log('splitIntoA7: after toBlob')
      // Calculate position in the grid (0-7)
      const position = row * cols + col
      // Get the page number based on ordering mode
      const pageNumber = getPageNumber(position, isPortrait, isFirstPage, currentOrderingMode)

      console.log('splitIntoA7: pushing piece', pageNumber)
      pieces.push({
        filename: `${baseName}_${pageNumber}.jpg`,
        dataUrl: dataUrl,
        canvas: finalCanvas
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
    const originalName = selectedFiles.value[0].name.replace(/\.\w+$/, '')
    link.download = `${originalName}_images.zip`

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
