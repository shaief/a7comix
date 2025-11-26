<template>
  <div class="container">
    <div class="app-nav">
      <a href="index.html" class="nav-link">📄 A7 Splitter</a>
      <span class="nav-separator">|</span>
      <span class="nav-current">📖 Pager</span>
    </div>
    <h1>📖 JPG Pager with Bleeds</h1>
    <p class="subtitle">Upload JPG files, select booklet mode, and add bleeds for printing</p>

    <!-- Upload Area -->
    <div
      v-if="!selectedFiles.length"
      class="upload-area"
      :class="{ dragging: isDragging }"
      @click="triggerFileInput"
      @dragover.prevent="isDragging = true"
      @dragleave.prevent="isDragging = false"
      @drop.prevent="handleDrop"
    >
      <div class="upload-icon">📎</div>
      <div class="upload-text">Click to upload or drag and drop</div>
      <div class="upload-hint">Supports multiple JPG files</div>
      <input
        ref="fileInput"
        type="file"
        class="file-input"
        accept="image/jpeg,image/jpg"
        multiple
        @change="handleFileSelect"
      >
    </div>

    <!-- File Info & Settings -->
    <div v-if="selectedFiles.length && !processedImages.length" class="file-info">
      <div class="file-header">
        <div>
          <h3>Selected Files: {{ selectedFiles.length }}</h3>
          <div class="file-list">
            <div v-for="(file, idx) in sortedFiles" :key="idx" class="file-item">
              {{ idx + 1 }}. {{ file.name }}
            </div>
          </div>
        </div>
        <button
          class="btn btn-secondary btn-small"
          @click="reset"
          :disabled="processing"
        >
          Choose Different Files
        </button>
      </div>

      <!-- Modern Settings Controls -->
      <div class="settings-panel">
        <!-- Page Size Controls -->
        <div class="setting-card">
          <div class="setting-header">
            <span class="setting-icon">📏</span>
            <span class="setting-title">Output Page Size</span>
          </div>
          <div class="chip-group">
            <button
              class="chip"
              :class="{ active: pageSize === 'a7' }"
              @click="setPageSize('a7')"
              :disabled="processing"
            >
              A7 (74×105mm)
            </button>
            <button
              class="chip"
              :class="{ active: pageSize === 'a6' }"
              @click="setPageSize('a6')"
              :disabled="processing"
            >
              A6 (105×148mm)
            </button>
            <button
              class="chip"
              :class="{ active: pageSize === 'a5' }"
              @click="setPageSize('a5')"
              :disabled="processing"
            >
              A5 (148×210mm)
            </button>
            <button
              class="chip"
              :class="{ active: pageSize === 'a4' }"
              @click="setPageSize('a4')"
              :disabled="processing"
            >
              A4 (210×297mm)
            </button>
          </div>
          <div class="setting-description">
            Images will be resized to fit within the selected page size (bleeds included)
          </div>
        </div>

        <!-- Booklet Mode Controls -->
        <div class="setting-card">
          <div class="setting-header">
            <span class="setting-icon">📖</span>
            <span class="setting-title">Booklet Mode</span>
          </div>
          <div class="chip-group">
            <button
              class="chip"
              :class="{ active: bookletMode === 'none' }"
              @click="setBookletMode('none')"
              :disabled="processing"
            >
              No Booklet (3mm all sides)
            </button>
            <button
              class="chip"
              :class="{ active: bookletMode === 'rtl' }"
              @click="setBookletMode('rtl')"
              :disabled="processing"
            >
              RTL Booklet
            </button>
            <button
              class="chip"
              :class="{ active: bookletMode === 'ltr' }"
              @click="setBookletMode('ltr')"
              :disabled="processing"
            >
              LTR Booklet
            </button>
          </div>
          <div class="setting-description">
            <span v-if="bookletMode === 'none'">All pages will have 3mm bleed on all sides</span>
            <span v-else-if="bookletMode === 'rtl'">Odd pages: 5mm right + 3mm others | Even pages: 5mm left + 3mm others</span>
            <span v-else>Odd pages: 5mm left + 3mm others | Even pages: 5mm right + 3mm others</span>
          </div>
        </div>

        <!-- Output File Name Controls -->
        <div class="setting-card">
          <div class="setting-header">
            <span class="setting-icon">📝</span>
            <span class="setting-title">Output File Name</span>
          </div>
          <div class="file-name-input-container">
            <input
              type="text"
              v-model="outputBaseName"
              :disabled="processing"
              placeholder="Enter base name for output files"
              class="file-name-input"
            >
          </div>
          <div class="setting-description">
            Leave empty to auto-derive from source files. Output: [base name]_1.jpg, [base name]_2.jpg, etc.
          </div>
        </div>
      </div>

      <div class="button-group">
        <button
          class="btn btn-primary"
          @click="processImages"
          :disabled="processing"
        >
          {{ processing ? 'Processing...' : 'Add Bleeds to Images' }}
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
      <button class="btn btn-primary" @click="downloadAsPdf">
        Download as PDF
      </button>
      <button class="btn btn-secondary" @click="reset">
        Upload Different Files
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
import { ref, computed } from 'vue'
import JSZip from 'jszip'
import { jsPDF } from 'jspdf'

// State
const selectedFiles = ref([])
const fileInput = ref(null)
const isDragging = ref(false)
const processing = ref(false)
const progress = ref(0)
const processedImages = ref([])
const statusMessage = ref('')
const statusType = ref('info')
const bookletMode = ref('rtl') // 'none', 'rtl', 'ltr'
const pageSize = ref('a7') // 'a7', 'a6', 'a5', 'a4'
const outputBaseName = ref('') // Custom base name for output files

// Page size dimensions in mm (width x height for portrait orientation)
const pageSizes = {
  a7: { width: 74, height: 105 },
  a6: { width: 105, height: 148 },
  a5: { width: 148, height: 210 },
  a4: { width: 210, height: 297 }
}

// Computed: Sort files by numeric value at end of filename
const sortedFiles = computed(() => {
  return [...selectedFiles.value].sort((a, b) => {
    // Extract number from end of filename (before extension)
    const getNumber = (filename) => {
      const nameWithoutExt = filename.replace(/\.\w+$/, '')
      const match = nameWithoutExt.match(/(\d+)$/)
      return match ? parseInt(match[1], 10) : 0
    }

    const numA = getNumber(a.name)
    const numB = getNumber(b.name)

    // If both have numbers, sort numerically
    if (numA !== 0 || numB !== 0) {
      return numA - numB
    }

    // Otherwise fall back to alphabetical
    return a.name.localeCompare(b.name)
  })
})

// File handling
const triggerFileInput = () => {
  fileInput.value.click()
}

const handleFileSelect = (event) => {
  const files = Array.from(event.target.files)
  if (files.length > 0) {
    selectedFiles.value = files
    processedImages.value = []
    statusMessage.value = ''
    bookletMode.value = 'rtl'
    pageSize.value = 'a7'
    // Extract base name from first file, removing trailing number and extension
    const firstName = files[0].name.replace(/\.\w+$/, '').replace(/_?\d+$/, '')
    outputBaseName.value = firstName
  }
}

const handleDrop = (event) => {
  isDragging.value = false
  const files = Array.from(event.dataTransfer.files)
  if (files.length > 0) {
    selectedFiles.value = files
    processedImages.value = []
    statusMessage.value = ''
    bookletMode.value = 'rtl'
    pageSize.value = 'a7'
    // Extract base name from first file, removing trailing number and extension
    const firstName = files[0].name.replace(/\.\w+$/, '').replace(/_?\d+$/, '')
    outputBaseName.value = firstName
  }
}

const reset = () => {
  selectedFiles.value = []
  processedImages.value = []
  statusMessage.value = ''
  progress.value = 0
  bookletMode.value = 'rtl'
  pageSize.value = 'a7'
  outputBaseName.value = ''
  if (fileInput.value) {
    fileInput.value.value = ''
  }
}

// Page size control
const setPageSize = (size) => {
  pageSize.value = size
}

// Booklet mode control
const setBookletMode = (mode) => {
  bookletMode.value = mode
}

// Convert mm to pixels at 300 DPI
const mmToPx = (mm) => {
  const dpi = 300
  return Math.round((mm / 25.4) * dpi)
}

// Get bleed values based on booklet mode and page number
const getBleedValues = (pageIndex, mode) => {
  const bleed3mm = mmToPx(3)
  const bleed5mm = mmToPx(5)

  // Page index is 0-based, but page numbers are 1-based
  const pageNumber = pageIndex + 1
  const isOddPage = pageNumber % 2 === 1

  if (mode === 'none') {
    // 3mm on all sides
    return { top: bleed3mm, right: bleed3mm, bottom: bleed3mm, left: bleed3mm }
  } else if (mode === 'rtl') {
    // RTL: odd pages get 5mm right, even pages get 5mm left
    if (isOddPage) {
      return { top: bleed3mm, right: bleed5mm, bottom: bleed3mm, left: bleed3mm }
    } else {
      return { top: bleed3mm, right: bleed3mm, bottom: bleed3mm, left: bleed5mm }
    }
  } else { // ltr
    // LTR: odd pages get 5mm left, even pages get 5mm right
    if (isOddPage) {
      return { top: bleed3mm, right: bleed3mm, bottom: bleed3mm, left: bleed5mm }
    } else {
      return { top: bleed3mm, right: bleed5mm, bottom: bleed3mm, left: bleed3mm }
    }
  }
}

// Image processing
const processImages = async () => {
  if (!selectedFiles.value.length) return

  processing.value = true
  progress.value = 0
  processedImages.value = []
  statusMessage.value = 'Processing images...'
  statusType.value = 'info'

  try {
    const sorted = sortedFiles.value
    const total = sorted.length
    const currentMode = bookletMode.value
    const currentPageSize = pageSize.value
    const currentBaseName = outputBaseName.value

    for (let i = 0; i < total; i++) {
      const file = sorted[i]
      const processedImage = await addBleedToImage(file, i, currentMode, currentPageSize, currentBaseName)
      processedImages.value.push(processedImage)
      progress.value = Math.round(((i + 1) / total) * 100)
    }

    statusMessage.value = `Successfully processed ${total} images!`
    statusType.value = 'success'
  } catch (error) {
    console.error('Error processing images:', error)
    statusMessage.value = `Error: ${error.message}`
    statusType.value = 'error'
  } finally {
    processing.value = false
  }
}

const addBleedToImage = (file, pageIndex, mode, pageSizeKey, baseName = '') => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()

    reader.onload = (e) => {
      const img = new Image()
      img.onload = () => {
        // Get bleed values for this page
        const bleeds = getBleedValues(pageIndex, mode)

        // Get target page size dimensions in mm
        const targetSize = pageSizes[pageSizeKey]

        // Convert page size from mm to pixels at 300 DPI
        const pageWidthPx = mmToPx(targetSize.width)
        const pageHeightPx = mmToPx(targetSize.height)

        // Calculate available space for content (page size minus bleeds)
        const contentWidth = pageWidthPx - bleeds.left - bleeds.right
        const contentHeight = pageHeightPx - bleeds.top - bleeds.bottom

        // Calculate scale factor to fit image within content area while maintaining aspect ratio
        const scaleX = contentWidth / img.width
        const scaleY = contentHeight / img.height
        const scale = Math.min(scaleX, scaleY)

        // Calculate scaled image dimensions
        const scaledWidth = img.width * scale
        const scaledHeight = img.height * scale

        // Create canvas with final page size
        const canvas = document.createElement('canvas')
        canvas.width = pageWidthPx
        canvas.height = pageHeightPx

        const ctx = canvas.getContext('2d')
        ctx.imageSmoothingEnabled = true
        ctx.imageSmoothingQuality = 'high'

        // Fill with white background
        ctx.fillStyle = 'white'
        ctx.fillRect(0, 0, canvas.width, canvas.height)

        // Center the scaled image within the content area
        const x = bleeds.left + (contentWidth - scaledWidth) / 2
        const y = bleeds.top + (contentHeight - scaledHeight) / 2

        // Draw the scaled image
        ctx.drawImage(img, x, y, scaledWidth, scaledHeight)

        // Convert to blob
        canvas.toBlob(
          (blob) => {
            const dataUrl = URL.createObjectURL(blob)

            // Extract number from original filename
            const originalName = file.name.replace(/\.\w+$/, '')
            const numberMatch = originalName.match(/_?(\d+)$/)
            const fileNumber = numberMatch ? numberMatch[1] : (pageIndex + 1)

            // Determine base name: use custom if provided, otherwise derive from file name
            let baseNameToUse
            if (baseName) {
              baseNameToUse = baseName
            } else {
              // Extract base name from file (remove trailing number)
              baseNameToUse = originalName.replace(/_?\d+$/, '')
            }

            const filename = `${baseNameToUse}_${fileNumber}.jpg`

            resolve({
              filename: filename,
              dataUrl: dataUrl,
              canvas: canvas
            })
          },
          'image/jpeg',
          0.95
        )
      }
      img.onerror = () => reject(new Error(`Failed to load image: ${file.name}`))
      img.src = e.target.result
    }

    reader.onerror = () => reject(new Error(`Failed to read file: ${file.name}`))
    reader.readAsDataURL(file)
  })
}

// Download functionality - create and download as ZIP
const downloadAllAsZip = async () => {
  try {
    statusMessage.value = 'Creating ZIP file...'
    statusType.value = 'info'

    const zip = new JSZip()

    // Add each image to the zip file
    for (const img of processedImages.value) {
      const response = await fetch(img.dataUrl)
      const blob = await response.blob()
      zip.file(img.filename, blob)
    }

    // Generate zip file
    const zipBlob = await zip.generateAsync({ type: 'blob' })

    // Create download link for the zip file
    const url = URL.createObjectURL(zipBlob)
    const link = document.createElement('a')
    link.href = url

    // Use custom base name if provided, otherwise use first file's name
    const baseName = outputBaseName.value || sortedFiles.value[0].name.replace(/\.\w+$/, '').replace(/_?\d+$/, '')
    link.download = `${baseName}_bleeds.zip`

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

// Download functionality - create and download as PDF
const downloadAsPdf = async () => {
  try {
    statusMessage.value = 'Creating PDF file...'
    statusType.value = 'info'

    if (processedImages.value.length === 0) {
      throw new Error('No images to export')
    }

    // Get dimensions from first image's canvas
    const firstImage = processedImages.value[0]
    const canvas = firstImage.canvas
    const widthMm = (canvas.width / 300) * 25.4  // Convert pixels at 300 DPI to mm
    const heightMm = (canvas.height / 300) * 25.4

    // Create PDF with custom page size matching the image dimensions
    const pdf = new jsPDF({
      orientation: widthMm > heightMm ? 'landscape' : 'portrait',
      unit: 'mm',
      format: [widthMm, heightMm]
    })

    // Add each image to the PDF
    for (let i = 0; i < processedImages.value.length; i++) {
      const img = processedImages.value[i]

      // Add new page for all images except the first
      if (i > 0) {
        pdf.addPage([widthMm, heightMm], widthMm > heightMm ? 'landscape' : 'portrait')
      }

      // Convert canvas to image data
      const imgData = img.canvas.toDataURL('image/jpeg', 0.95)

      // Add image to PDF (fill entire page)
      pdf.addImage(imgData, 'JPEG', 0, 0, widthMm, heightMm)
    }

    // Use custom base name if provided, otherwise use first file's name
    const baseName = outputBaseName.value || sortedFiles.value[0].name.replace(/\.\w+$/, '').replace(/_?\d+$/, '')
    const filename = `${baseName}_bleeds.pdf`

    // Save the PDF
    pdf.save(filename)

    statusMessage.value = 'PDF file downloaded successfully!'
    statusType.value = 'success'
  } catch (error) {
    console.error('Error creating PDF:', error)
    statusMessage.value = `Error creating PDF: ${error.message}`
    statusType.value = 'error'
  }
}
</script>

<style scoped>
.file-list {
  max-height: 200px;
  overflow-y: auto;
  margin-top: 0.5rem;
  padding: 0.5rem;
  background: rgba(0, 0, 0, 0.02);
  border-radius: 8px;
}

.file-item {
  padding: 0.25rem 0;
  font-size: 0.9rem;
  color: #555;
}

.setting-description {
  margin-top: 0.75rem;
  font-size: 0.85rem;
  color: #666;
  text-align: center;
  padding: 0.5rem;
  background: rgba(102, 126, 234, 0.05);
  border-radius: 6px;
}
</style>
