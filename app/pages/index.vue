<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { createWorker, OEM, PSM } from 'tesseract.js'
import { parse } from 'mrz'
import * as pdfjsLib from 'pdfjs-dist'

console.log('Index page setup started')

// Set worker source for pdfjs-dist
// We use a local worker file copied to public/ directory to avoid CDN and build issues
onMounted(() => {
  pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs'
})

interface MrzResult {
  documentType: string
  documentNumber: string
  series?: string
  firstName: string
  lastName: string
  nationality: string
  birthDate: string
  sex: string
  expirationDate: string
  personal_number?: string
  issuingCountry: string
  valid: boolean
  details: any[]
  unifiedJson?: any
}

const file = ref<File | null>(null)
const imagePreview = ref<string | null>(null)
const isProcessing = ref(false)
const progress = ref(0)
const progressStatus = ref('')
const rawMrzText = ref('')
const parsedResult = ref<MrzResult | null>(null)
const error = ref<string | null>(null)

const supportedFormats = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf']

const convertPdfToImage = async (file: File): Promise<string> => {
  const arrayBuffer = await file.arrayBuffer()
  const loadingTask = pdfjsLib.getDocument({ data: arrayBuffer })
  const pdf = await loadingTask.promise
  const page = await pdf.getPage(1) // Get first page
  
  // High scale for better OCR results
  const scale = 2.0
  const viewport = page.getViewport({ scale })
  
  const canvas = document.createElement('canvas')
  const context = canvas.getContext('2d')
  
  if (!context) throw new Error('Could not create canvas context')
  
  canvas.height = viewport.height
  canvas.width = viewport.width
  
  await page.render({
    canvasContext: context,
    viewport: viewport
  }).promise
  
  return canvas.toDataURL('image/png')
}

const handleFileSelect = async (event: Event) => {
  const target = event.target as HTMLInputElement
  const selectedFile = target.files?.[0]
  
  if (!selectedFile) return
  
  if (!supportedFormats.includes(selectedFile.type)) {
    error.value = 'Unsupported file format. Please use JPEG, PNG, WebP, or PDF.'
    return
  }
  
  reset()
  file.value = selectedFile
  
  try {
    if (selectedFile.type === 'application/pdf') {
      progressStatus.value = 'Rendering PDF...'
      isProcessing.value = true
      
      try {
        const imageData = await convertPdfToImage(selectedFile)
        imagePreview.value = imageData
      } catch (err: any) {
        error.value = `Failed to render PDF: ${err.message}`
      } finally {
        isProcessing.value = false
        progressStatus.value = ''
      }
    } else {
      // Create preview for images
      const reader = new FileReader()
      reader.onload = (e) => {
        imagePreview.value = e.target?.result as string
      }
      reader.readAsDataURL(selectedFile)
    }
  } catch (err: any) {
    error.value = `Error loading file: ${err.message}`
  }
}

const handleDrop = (event: DragEvent) => {
  event.preventDefault()
  const droppedFile = event.dataTransfer?.files?.[0]
  
  if (droppedFile) {
    const fakeEvent = {
      target: { files: [droppedFile] }
    } as unknown as Event
    handleFileSelect(fakeEvent)
  }
}

const handleDragOver = (event: DragEvent) => {
  event.preventDefault()
}

const extractMrzFromText = (text: string): string[] => {
  // MRZ lines are typically 30 (TD1), 36 (TD2), or 44 (TD3/Passport) characters
  // Cleaning: remove spaces within valid line patterns if they caused mostly by OCR gaps
  const lines = text.split('\n').map(line => line.trim())
  
  // Filter lines that look like MRZ (contain < and alphanumeric)
  const mrzLines = lines.filter(line => {
    // Basic heuristics for MRZ line
    // Must contain some << or many Uppercase/Numbers
    // Remove spaces to check length
    const cleanLine = line.replace(/\s/g, '')
    const hasMrzChars = /[A-Z0-9<]/.test(cleanLine)
    // Relaxed length check to capture noisy lines (e.g. 46 chars instead of 44)
    const isValidLength = cleanLine.length >= 20 && cleanLine.length <= 100 
    const chevronCount = (cleanLine.match(/</g) || []).length
    
    // MRZ lines usually have multiple chevrons
    return hasMrzChars && isValidLength && (chevronCount >= 2 || cleanLine.length === 30 || cleanLine.length === 36 || cleanLine.length === 44)
  })
  
  // Map back to space-stripped for the parser
  return mrzLines.map(l => l.replace(/\s/g, '').toUpperCase())
}

const processMrz = async () => {
  if (!file.value && !imagePreview.value) {
    error.value = 'Please select a file first'
    return
  }
  
  isProcessing.value = true
  progress.value = 0
  error.value = null
  parsedResult.value = null
  rawMrzText.value = ''
  
  try {
    progressStatus.value = 'Initializing OCR engine...'
    
    // Use 'eng' as base, 'ocrb' if available, but 'eng' is usually sufficient for MRZ with whitelist
    const worker = await createWorker('eng', OEM.LSTM_ONLY, {
      logger: (m) => {
        if (m.status === 'recognizing text') {
          progress.value = Math.round(m.progress * 100)
          progressStatus.value = `Processing: ${progress.value}%`
        }
      }
    })
    
    // Configure for MRZ recognition
    await worker.setParameters({
      tessedit_pageseg_mode: PSM.SINGLE_BLOCK,
      // MRZ characters: A-Z, 0-9, <
      tessedit_char_whitelist: 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789<',
    })
    
    progressStatus.value = 'Recognizing text...'
    
    // If it's a PDF or we already have a preview (which is dataURL), use that.
    // Tesseract works well with DataURL.
    let input: File | string = imagePreview.value || file.value!
    
    const { data: { text } } = await worker.recognize(input)
    
    await worker.terminate()
    
    progressStatus.value = 'Parsing MRZ...'
    
    // --- Advanced Parsing Logic Start ---

    // --- Integration with 'mrz' library ---

    // 1. Cleaning function
    const cleanMrzLine = (line: string): string => {
        let cleaned = line.toUpperCase()

        // 1. Fix Start: P[SLCK]... -> P<... or I[SLCK]... -> I<...
        if (/^P[SLCK][A-Z<]{3}/.test(cleaned)) {
            cleaned = 'P<' + cleaned.substring(2)
        }
        if (/^I[SLCK][A-Z<]{3}/.test(cleaned)) {
             cleaned = 'I<' + cleaned.substring(2)
        }

        // 2. Fix Separators: <[CL] followed by letter -> <<
        // Use a safe regex that avoids matching <<C or <<L (already separators)
        cleaned = cleaned.replace(/(^|[^<])<[CL](?=[A-Z])/g, '$1<<')

        // 3. Fix Tail Fillers: Replace [L,C,K] at end of line with <
        const match = cleaned.match(/[LCK<]{3,}$/)
        if (match) {
            const tail = match[0]
            if (/[LCK]/.test(tail)) {
                const newTail = tail.replace(/[LCK]/g, '<')
                cleaned = cleaned.substring(0, match.index) + newTail
            }
        }
        
        // 4. Fix Length: Pad specific known types if they are short (OCR cut off end)
        // TD-3 starts with P< and must be 44 chars
        if (cleaned.startsWith('P<') && cleaned.length < 44 && cleaned.length > 30) {
            cleaned = cleaned.padEnd(44, '<')
        }
        // TD-1 starts with I< (or A, C) and usually 30. Our cleaner fixes I... -> I<
        if (cleaned.startsWith('I<') && cleaned.length < 30 && cleaned.length > 20) {
            cleaned = cleaned.padEnd(30, '<')
        }

        return cleaned
    }

    // 2. Extract and Clean Lines
    const mrzLinesRaw = extractMrzFromText(text)
    
    // Apply cleaning to all extracted lines
    const mrzLines = mrzLinesRaw.map(l => {
        try {
            return cleanMrzLine(l)
        } catch (e) {
            console.error('Cleaning failed for line:', l, e)
            return l
        }
    })
    
    if (mrzLines.length < 2) {
      error.value = 'Could not detect MRZ in the document. Please ensure the MRZ area is clearly visible and crisp.'
      rawMrzText.value = text
      isProcessing.value = false
      return
    }
    
    // 3. Parsing logic using 'mrz' library
    let result: any = null
    
    // Helper to attempt parsing a set of lines
    const tryParseMRZ = (lines: string[]) => {
        try {
            return parse(lines)
        } catch (e) {
            return null
        }
    }
    
    // Helper to generate sliding window candidates for a line
    const getCandidates = (line: string, targetLen: number): string[] => {
        const candidates: string[] = []
        if (line.length === targetLen) candidates.push(line)
        if (line.length > targetLen) {
            for (let i = 0; i <= line.length - targetLen; i++) {
                candidates.push(line.substring(i, i + targetLen))
            }
        }
        return candidates
    }

    // Strategy: Try last 2 lines (TD3), last 3 lines (TD1) with sliding window
    
    // Try TD-3 (Target 44 chars)
    if (!result && mrzLines.length >= 2) {
        const last2 = mrzLines.slice(-2)
        const l1Candidates = getCandidates(last2[0], 44)
        const l2Candidates = getCandidates(last2[1], 44)
        
        for (const c1 of l1Candidates) {
            for (const c2 of l2Candidates) {
                const res = tryParseMRZ([c1, c2])
                if (res && res.valid) {
                    result = res
                    break
                }
                if (res && !result) result = res
            }
            if (result?.valid) break
        }
    }

    // Try TD-1 (Target 30 chars)
    if ((!result || !result.valid) && mrzLines.length >= 3) {
        const last3 = mrzLines.slice(-3)
        const l1C = getCandidates(last3[0], 30)
        const l2C = getCandidates(last3[1], 30)
        const l3C = getCandidates(last3[2], 30)

        for (const c1 of l1C) {
            for (const c2 of l2C) {
                for (const c3 of l3C) {
                   const res = tryParseMRZ([c1, c2, c3])
                   if (res && res.valid) {
                       result = res
                       break
                   }
                   if (res && !result) result = res
                }
                if (result?.valid) break
            }
            if (result?.valid) break
        }
    }
    
    // Fallback: Just try what we have if selection failed or wasn't applicable
    if (!result && mrzLines.length > 0) {
        result = tryParseMRZ(mrzLines)
    }

    rawMrzText.value = mrzLines.join('\n')
    
    if (result) {
        parsedResult.value = {
            documentType: result.format,
            documentNumber: result.fields.documentNumber,
            firstName: result.fields.firstName,
            lastName: result.fields.lastName,
            nationality: result.fields.nationality,
            birthDate: result.fields.birthDate,
            sex: result.fields.sex,
            expirationDate: result.fields.expirationDate,
            issuingCountry: result.fields.issuingState || result.fields.nationality,
            personal_number: result.fields.personalNumber,
            valid: result.valid,
            details: result.details || [],
            unifiedJson: result
        } as any
    } else {
         error.value = `MRZ detected but parsing failed. Could not fit to TD-3 (44) or TD-1 (30) standards.`
    }
    
    progressStatus.value = 'Complete!'
    
  } catch (err: any) {
    error.value = `Processing failed: ${err.message}`
  } finally {
    isProcessing.value = false
  }
}

const reset = () => {
  file.value = null
  imagePreview.value = null
  parsedResult.value = null
  rawMrzText.value = ''
  error.value = null
  progress.value = 0
  progressStatus.value = ''
}

const formatDate = (dateStr: string) => {
    return dateStr // Format is already done in parser
}
</script>

<template>
  <div class="mrz-scanner">
    <h1 class="title">MRZ Scanner</h1>
    <p class="subtitle">Upload a passport, ID card, or visa to extract MRZ data</p>
    
    <!-- File Upload Area -->
    <div 
      class="upload-area"
      :class="{ 'has-file': file }"
      @drop="handleDrop"
      @dragover="handleDragOver"
    >
      <input 
        type="file" 
        id="file-input"
        accept=".jpg,.jpeg,.png,.webp,.pdf,application/pdf,image/*"
        @change="handleFileSelect"
        class="file-input"
      />
      <label for="file-input" class="upload-label">
        <div v-if="!file" class="upload-prompt">
          <svg class="upload-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/>
            <polyline points="17 8 12 3 7 8"/>
            <line x1="12" y1="3" x2="12" y2="15"/>
          </svg>
          <span class="upload-text">Drag & drop or click to upload</span>
          <span class="upload-formats">Supports: JPEG, PNG, WebP, PDF</span>
        </div>
        <div v-else class="file-info">
          <span class="file-name">{{ file.name }}</span>
          <span class="file-size">{{ (file.size / 1024).toFixed(1) }} KB</span>
        </div>
      </label>
    </div>
    
    <!-- Image Preview -->
    <div v-if="imagePreview" class="preview-container">
      <img :src="imagePreview" alt="Document preview" class="preview-image" />
    </div>
    
    <!-- Action Buttons -->
    <div class="actions">
      <button 
        class="btn btn-primary" 
        @click="processMrz"
        :disabled="(!file && !imagePreview) || isProcessing"
      >
        <span v-if="isProcessing">Processing...</span>
        <span v-else>Scan MRZ</span>
      </button>
      <button 
        class="btn btn-secondary" 
        @click="reset"
        :disabled="isProcessing"
      >
        Reset
      </button>
    </div>
    
    <!-- Progress Bar -->
    <div v-if="isProcessing" class="progress-container">
      <div class="progress-bar">
        <div class="progress-fill" :style="{ width: `${progress}%` }"></div>
      </div>
      <span class="progress-text">{{ progressStatus }}</span>
    </div>
    
    <!-- Error Message -->
    <div v-if="error" class="error-message">
      <svg class="error-icon" viewBox="0 0 24 24" fill="currentColor">
        <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-2h2v2zm0-4h-2V7h2v6z"/>
      </svg>
      {{ error }}
    </div>
    
    <!-- Results -->
    <div v-if="parsedResult" class="results">
      <h2 class="results-title">
        <span>Extracted Data</span>
        <span :class="['validity-badge', parsedResult.valid ? 'valid' : 'invalid']">
          {{ parsedResult.valid ? '✓ Valid' : '✗ Invalid' }}
        </span>
      </h2>
      
      <div class="result-grid">
        <div class="result-item">
          <label>Document Type</label>
          <span>{{ parsedResult.documentType }}</span>
        </div>
        <div class="result-item">
          <label>Document Number</label>
          <span>{{ parsedResult.documentNumber }}</span>
        </div>
        <div class="result-item">
          <label>Series</label>
          <span>{{ parsedResult.series || parsedResult.documentNumber.substring(0,2) }}</span>
        </div>
        <div class="result-item">
          <label>Last Name</label>
          <span>{{ parsedResult.lastName }}</span>
        </div>
        <div class="result-item">
          <label>First Name</label>
          <span>{{ parsedResult.firstName }}</span>
        </div>
        <div class="result-item">
          <label>Nationality</label>
          <span>{{ parsedResult.nationality }}</span>
        </div>
        <div class="result-item">
          <label>Personal Number</label>
          <span>{{ parsedResult.personal_number || '-' }}</span>
        </div>
        <div class="result-item">
          <label>Date of Birth</label>
          <span>{{ formatDate(parsedResult.birthDate) }}</span>
        </div>
        <div class="result-item">
          <label>Expiration Date</label>
          <span>{{ formatDate(parsedResult.expirationDate) }}</span>
        </div>
        <div class="result-item">
          <label>Sex</label>
          <span>{{ parsedResult.sex }}</span>
        </div>
      </div>

       <!-- JSON Output Section -->
       <div class="unified-json" v-if="parsedResult.unifiedJson">
        <h3>Unified JSON Output</h3>
        <pre>{{ JSON.stringify(parsedResult.unifiedJson, null, 2) }}</pre>
      </div>
    </div>
    
    <!-- Raw MRZ Text -->
    <div v-if="rawMrzText" class="raw-mrz">
      <h3>Raw MRZ Text</h3>
      <pre>{{ rawMrzText }}</pre>
    </div>
  </div>
</template>

<style scoped>
.mrz-scanner {
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
  font-family: 'Segoe UI', system-ui, sans-serif;
}

.title {
  font-size: 2.5rem;
  font-weight: 700;
  text-align: center;
  margin-bottom: 0.5rem;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.subtitle {
  text-align: center;
  color: #6b7280;
  margin-bottom: 2rem;
}

.upload-area {
  border: 2px dashed #d1d5db;
  border-radius: 12px;
  padding: 2rem;
  text-align: center;
  transition: all 0.3s ease;
  background: #f9fafb;
}

.upload-area:hover {
  border-color: #667eea;
  background: #f0f4ff;
}

.upload-area.has-file {
  border-color: #667eea;
  border-style: solid;
  background: #f0f4ff;
}

.file-input {
  display: none;
}

.upload-label {
  cursor: pointer;
  display: block;
}

.upload-icon {
  width: 48px;
  height: 48px;
  color: #9ca3af;
  margin: 0 auto 1rem;
}

.upload-text {
  display: block;
  font-size: 1.1rem;
  color: #374151;
  margin-bottom: 0.5rem;
}

.upload-formats {
  display: block;
  font-size: 0.875rem;
  color: #9ca3af;
}

.file-info {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.file-name {
  font-weight: 600;
  color: #374151;
}

.file-size {
  font-size: 0.875rem;
  color: #6b7280;
}

.preview-container {
  margin-top: 1.5rem;
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

.preview-image {
  width: 100%;
  max-height: 400px;
  object-fit: contain;
  background: #1f2937;
}

.actions {
  display: flex;
  gap: 1rem;
  justify-content: center;
  margin-top: 1.5rem;
  padding: 0.75rem 2rem;
}

.btn {
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
  border: none;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

.btn-primary:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.btn-secondary {
  background: #e5e7eb;
  color: #374151;
}

.btn-secondary:hover:not(:disabled) {
  background: #d1d5db;
}

.progress-container {
  margin-top: 1.5rem;
  text-align: center;
}

.progress-bar {
  height: 8px;
  background: #e5e7eb;
  border-radius: 4px;
  overflow: hidden;
  width: 100%;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  transition: width 0.3s ease;
}

.progress-text {
  display: block;
  margin-top: 0.5rem;
  font-size: 0.875rem;
  color: #6b7280;
}

.error-message {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-top: 1.5rem;
  padding: 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  color: #dc2626;
}

.error-icon {
  width: 20px;
  height: 20px;
  flex-shrink: 0;
}

.results {
  margin-top: 2rem;
  padding: 1.5rem;
  background: white;
  border-radius: 12px;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

.results-title {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
  font-size: 1.25rem;
  font-weight: 600;
}

.validity-badge {
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.875rem;
  font-weight: 600;
}

.validity-badge.valid {
  background: #d1fae5;
  color: #059669;
}

.validity-badge.invalid {
  background: #fee2e2;
  color: #dc2626;
}

.result-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1rem;
}

.result-item {
  padding: 1rem;
  background: #f9fafb;
  border-radius: 8px;
}

.result-item label {
  display: block;
  font-size: 0.75rem;
  text-transform: uppercase;
  color: #6b7280;
  margin-bottom: 0.25rem;
  font-weight: 600;
}

.result-item span {
  font-size: 1rem;
  font-weight: 500;
  color: #1f2937;
  word-break: break-word;
}

.raw-mrz {
  margin-top: 1.5rem;
  padding: 1rem;
  background: #1f2937;
  border-radius: 8px;
}

.raw-mrz h3 {
  color: #9ca3af;
  font-size: 0.875rem;
  margin-bottom: 0.5rem;
}

.raw-mrz pre {
  color: #10b981;
  font-family: 'Courier New', monospace;
  font-size: 0.875rem;
  white-space: pre-wrap;
  word-break: break-all;
  margin: 0;
}

.unified-json {
  margin-top: 1.5rem;
  padding: 1rem;
  background: #282c34;
  border-radius: 8px;
  overflow-x: auto;
}

.unified-json h3 {
  color: #abb2bf;
  font-size: 0.875rem;
  margin-bottom: 0.5rem;
}

.unified-json pre {
  color: #e06c75;
  font-family: 'Courier New', monospace;
  font-size: 0.85rem;
  margin: 0;
}
</style>
