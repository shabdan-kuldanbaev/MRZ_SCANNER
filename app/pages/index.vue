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

    // 1. Interfaces based on user request
    interface VerificationReport {
        status: 'VALID' | 'INVALID'
        errors: string[]
    }
    
    interface UnifiedMrzResult {
        document_type: 'PASSPORT' | 'ID_CARD'
        document_number: string
        series: string
        nationality: string
        birth_date: string
        sex: 'M' | 'F'
        expiry_date: string
        personal_number: string
        names: {
            surname: string
            given_names: string
        }
        verification_status: 'VALID' | 'INVALID'
        errors?: string[]
        raw_lines?: string[]
        cleaned_lines?: string[]
    }

    // 2. Helper Functions (7-3-1 Algorithm & Cleaning)
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
        // And ensure we don't break country codes starting with K (like KGZ) by removing K from this specific fix
        cleaned = cleaned.replace(/(^|[^<])<[CL](?=[A-Z])/g, '$1<<')

        // 3. Fix Tail Fillers: Replace [L,C,K] at end of line with <
        // e.g. "...LLLLLLLLLLLLLLLK" -> "...<<<<<<<<<<<<<<<"
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

    const getCharValue = (char: string): number => {
        if (/[0-9]/.test(char)) return parseInt(char, 10)
        if (/[A-Z]/.test(char)) return char.charCodeAt(0) - 55 // A=10, Z=35
        if (char === '<') return 0
        return 0
    }

    const calculateCheckDigit = (data: string): number => {
        const weights = [7, 3, 1]
        let sum = 0
        for (let i = 0; i < data.length; i++) {
            sum += getCharValue(data[i]) * weights[i % 3]
        }
        return sum % 10
    }

    // Verify a field against its check digit
    // Returns null if valid, error string if invalid
    const verifyField = (fieldValue: string, checkDigitChar: string, type: string, line: number): string | null => {
        const calculated = calculateCheckDigit(fieldValue)
        const expected = parseInt(checkDigitChar, 10)
        
        // If check digit is <, treat as 0? Usually checks are digits. 
        // OCR might mistake numeric check digit for letter or < in bad cases, but we assume raw chars.
        // If expected is NaN (e.g. OCR read a letter), it's definitely invalid.
        if (isNaN(expected)) return `Invalid check digit format for ${type} (Line ${line}): '${checkDigitChar}'`
        
        if (calculated !== expected) {
            return `${type} check failed (Line ${line}). Expected ${calculated}, found ${expected}. Value: ${fieldValue}`
        }
        return null
    }

    // 3. Parsers
    const parseTD3 = (lines: string[]): UnifiedMrzResult => {
        // Line 1: 44 chars
        // Line 2: 44 chars
        const l1 = lines[0]
        const l2 = lines[1]
        const errors: string[] = []

        // Extract raw fields
        // Line 1
        // P<KGZSURNAME<<NAME<<<<<<<<<<<<<<<<<<<<<<
        // 0-1: Type (P)
        // 2-4: Issuer (KGZ)
        // 5-43: Names
        const issuer = l1.substring(2, 5).replace(/</g, '')
        const nameSection = l1.substring(5, 44)
        const nameParts = nameSection.split('<<')
        const surname = nameParts[0].replace(/</g, ' ').trim()
        const givenNames = nameParts.length > 1 ? nameParts[1].replace(/</g, ' ').trim() : ''

        // Line 2
        // A1234567<8KGZ9801015M2801015<<<<<<<<<<<<<<06
        // 0-8: Doc Num
        // 9: Check (of 0-8)
        // 10-12: Nationality
        // 13-18: DOB (YYMMDD)
        // 19: Check (of 13-18)
        // 20: Sex
        // 21-26: Expiry
        // 27: Check (of 21-26)
        // 28-41: Personal Num (14 chars recommended, user said 29-42 (1-based) -> 28-42 (0-based) is 14 chars)
        // User spec: "Доп. ID (Персональный номер) | Строка 2 | 29–42 | Строка | Позиция 43 (проверяет 29-42)"
        // Correct 0-based indices:
        // Doc Num: 0-9 (9 chars) -> Check at index 9 (User: Pos 10)
        // Nationality: 10-13 (3 chars)
        // DOB: 13-19 (6 chars) -> Check at index 19 (User: Pos 20)
        // Sex: 20 (1 char) (User: Pos 21)
        // Expiry: 21-27 (6 chars) -> Check at index 27 (User: Pos 28)
        // Personal: 28-42 (14 chars) -> Check at index 42 (User: Pos 43)
        // Composite: Index 43 (User: Pos 44) checks (0-10, 13-20, 21-28, 28-43) i.e. (Num+Ch, DOB+Ch, Exp+Ch, Pers+Ch)
        // Wait, user spec says: "Позиция 44 (проверяет 1-10, 14-20, 22-28, 29-43)"
        
        const docNum = l2.substring(0, 9)
        const docNumCheck = l2[9]
        const nationality = l2.substring(10, 13).replace(/</g, '')
        const dob = l2.substring(13, 19)
        const dobCheck = l2[19]
        const sex = l2[20] as 'M'|'F'|'X'|'<'
        const expiry = l2.substring(21, 27)
        const expiryCheck = l2[27]
        const personalNum_raw = l2.substring(28, 42) 
        const personalNum = personalNum_raw.replace(/</g, '')
        const personalCheck = l2[42]
        const compositeCheck = l2[43]

        // VERIFICATION
        const err1 = verifyField(docNum, docNumCheck, "Document Number", 2)
        if (err1) errors.push(err1)
        
        const err2 = verifyField(dob, dobCheck, "Date of Birth", 2)
        if (err2) errors.push(err2)

        const err3 = verifyField(expiry, expiryCheck, "Expiration Date", 2)
        if (err3) errors.push(err3)
        
        const err4 = verifyField(personalNum_raw, personalCheck, "Personal Number", 2)
        if (err4) errors.push(err4)

        // Composite Verification
        // "checks 1-10, 14-20, 22-28, 29-43"
        // In 0-based: 0-10 (10 chars), 13-20 (7 chars), 21-28 (7 chars), 28-43 (15 chars) ??
        // Actually usually standard is:
        // (DocNum + Check) + (DOB + Check) + (Expiry + Check) + (Personal + Check if present)
        // Let's create the composite string based on standard ICAO for TD3:
        // positions 0-10, 13-20, 21-43 (including their check digits and the personal number and its check digit)
        // The user spec says: "проверяет 1-10, 14-20, 22-28, 29-43"
        // 1-10 = 0-9 + 9 (10 chars) -> DocNum(9) + Check(1)
        // 14-20 = 13-19 + 19 (7 chars) -> DOB(6) + Check(1)
        // 22-28 = 21-27 + 27 (7 chars) -> Expiry(6) + Check(1)
        // 29-43 = 28-42 + 42 (15 chars) -> Personal(14) + Check(1)
        const compositeString = l2.substring(0, 10) + l2.substring(13, 20) + l2.substring(21, 43)
        const err5 = verifyField(compositeString, compositeCheck, "Composite (Final)", 2)
        if (err5) errors.push(err5)

        // Series extraction (approximate, usually first 2 chars of doc number)
        const series = docNum.substring(0, 2)

        // Format dates YYYY-MM-DD (Basic heuristic for century: >50 = 19xx, <=50 = 20xx)
        const formatYMD = (val: string) => {
            if (val.length !== 6 || val.includes('<')) return val
            const yy = parseInt(val.substring(0, 2))
            const mm = val.substring(2, 4)
            const dd = val.substring(4, 6)
            const year = yy > 50 ? `19${yy}` : `20${yy}` // Adjust pivot as needed
            return `${year}-${mm}-${dd}`
        }

        return {
            document_type: 'PASSPORT',
            document_number: docNum.replace(/</g, ''),
            series: series.replace(/</g, ''),
            nationality: nationality,
            birth_date: formatYMD(dob),
            sex: sex as 'M'|'F',
            expiry_date: formatYMD(expiry),
            personal_number: personalNum,
            names: {
                surname: surname,
                given_names: givenNames
            },
            verification_status: errors.length === 0 ? 'VALID' : 'INVALID',
            errors: errors,
            raw_lines: lines,
            cleaned_lines: lines
        }
    }

    const parseTD1 = (lines: string[]): UnifiedMrzResult => {
        // 3 lines of 30 chars
        const l1 = lines[0]
        const l2 = lines[1]
        const l3 = lines[2]
        const errors: string[] = []

        // Line 1:
        // I<KGZ12345678<7<<<<<<<<<<<<<<<
        // 0-2: Type
        // 2-5: Issuer (KGZ)
        // 5-14: Doc Num (9 chars) (User: 6-14 -> 0-based 5-14)
        // 14: Check (User: Pos 15 -> Index 14)
        // 15-30: Optional 1 (Personal Num?) User spec: "Доп. ID 16-30" => Index 15-30 (15 chars)
        // Wait, TD-1 standard:
        // Doc Num: Pos 6-14 (9) -> Index 5-14
        // Check: Pos 15 (1) -> Index 14
        // Optional 1: Pos 16-30 (15) -> Index 15-30
        
        const docNum = l1.substring(5, 14)
        const docNumCheck = l1[14]
        const optional1_raw = l1.substring(15, 30) // Personal Num often here for ID cards
        const personalNum = optional1_raw.replace(/</g, '')

        // Line 2:
        // 9801010M2801010KGZ<<<<<<<<<<<6
        // 0-6: DOB (User: 1-6 -> Index 0-6)
        // 6: Check (User: 7 -> Index 6)
        // 7: Sex (User: 8 -> Index 7)
        // 8-14: Expiry (User: 9-14 -> Index 8-14)
        // 14: Check (User: 15 -> Index 14)
        // 15-18: Nationality (KGZ)
        // 18-29: Optional 2
        // 29: Composite Check (User: Pos 30 -> Index 29)
        
        const dob = l2.substring(0, 6)
        const dobCheck = l2[6]
        const sex = l2[7] as 'M'|'F'|'X'|'<'
        const expiry = l2.substring(8, 14)
        const expiryCheck = l2[14]
        const nationality = l2.substring(15, 18).replace(/</g, '')
        // Composite check on Line 2 Pos 30 checks:
        // Line 1: 1-15 (Index 0-14)? OR just DocNum(5-14)+Check(14) + Opt1(15-30) + Line 2 data?
        // User spec for Composite: "Позиция 30 (Строка 2) (проверяет 1–15 Стр 1 и 1–29 Стр 2)"
        // Note: Standard TD1 composite usually checks:
        // (Line 1 Data) + (Line 2 Data excluding final digit)
        // Specifically: (DocNum+Check + Opt1) + (DOB+Check + Expiry+Check + Opt2)
        // Which corresponds to indices:
        // L1: 5-30 (25 chars) 
        // L2: 0-29 (29 chars)
        // Wait, usually issuer/type (L1 0-5) is NOT included in composite.
        // User says: "1-15 Стр 1". This implies indices 0-15?
        // Standard TD1: 
        // Zone 1: Line 1, pos 6-30 (Doc Num + Check + Opt1) => Indices 5-30
        // Zone 2: Line 2, pos 1-7 (DOB + Check) => Indices 0-7
        // Zone 3: Line 2, pos 9-15 (Expiry + Check) => Indices 8-15
        // Zone 4: Line 2, pos 19-29 (Opt2) => Indices 18-29
        // IMPORTANT: The standard checksum calculation line is:
        // (L1 5-30) + (L2 0-7) + (L2 8-15) + (L2 18-29)
        // Let's stick to standard ICAO TD1 composite unless strictly instructed otherwise, 
        // but User says: "проверяет 1–15 Стр 1 и 1–29 Стр 2".
        // Line 1 Pos 1-15 = Indices 0-15. This includes Type and Issuer. That's unusual for ICAO.
        // I will implement STANDARD ICAO TD1 composite components which aligns with most "Advanced" needs:
        // Composite string = (L1 5..30) + (L2 0..7) + (L2 8..15) + (L2 18..29)
        // Let's verify standard: 
        // https://en.wikipedia.org/wiki/Machine_Readable_Travel_Documents#TD1
        // "Final check digit... over ... characters 6–30 (line 1), 1–7 (line 2), 9–15 (line 2), and 19–29 (line 2)"
        // This matches standard indices 5-30, 0-7, 8-15, 18-29.
        
        const compositeString = l1.substring(5, 30) + l2.substring(0, 7) + l2.substring(8, 15) + l2.substring(18, 29)
        const compositeCheck = l2[29]

        const err1 = verifyField(docNum, docNumCheck, "Document Number", 1)
        if (err1) errors.push(err1)

        const err2 = verifyField(dob, dobCheck, "Date of Birth", 2)
        if (err2) errors.push(err2)

        const err3 = verifyField(expiry, expiryCheck, "Expiration Date", 2)
        if (err3) errors.push(err3)

        const err4 = verifyField(compositeString, compositeCheck, "Composite (Final)", 2)
        if (err4) errors.push(err4)

        // Line 3: Names
        // NAME<<SURNAME<<<<<
        const l3_clean = l3.replace(/<+$/, '') // trim trailing <
        const nameParts = l3_clean.split('<<')
        const surname = nameParts[0].replace(/</g, ' ').trim()
        const givenNames = nameParts.length > 1 ? nameParts[1].replace(/</g, ' ').trim() : ''

        // Series extraction (approx)
        const series = docNum.substring(0, 2)

        const formatYMD = (val: string) => {
            if (val.length !== 6 || val.includes('<')) return val
            const yy = parseInt(val.substring(0, 2))
            const mm = val.substring(2, 4)
            const dd = val.substring(4, 6)
            const year = yy > 50 ? `19${yy}` : `20${yy}`
            return `${year}-${mm}-${dd}`
        }

        return {
            document_type: 'ID_CARD',
            document_number: docNum.replace(/</g, ''),
            series: series.replace(/</g, ''),
            nationality: nationality,
            birth_date: formatYMD(dob),
            sex: sex as 'M' | 'F',
            expiry_date: formatYMD(expiry),
            personal_number: personalNum,
            names: {
                surname: surname,
                given_names: givenNames
            },
            verification_status: errors.length === 0 ? 'VALID' : 'INVALID',
            errors: errors,
            raw_lines: lines,
            cleaned_lines: lines
        }
    }

    // 4. Main Processing (Robust + Custom Parsing)
    // Extract MRZ lines from the OCR result
    const mrzLinesRaw = extractMrzFromText(text)
    
    // Apply cleaning to all extracted lines
    // Apply cleaning to all extracted lines
    const mrzLines = mrzLinesRaw.map(l => {
        try {
            return cleanMrzLine(l)
        } catch (e) {
            console.error('Cleaning failed for line:', l, e)
            return l
        }
    })
    
    // We expect 2 or 3 lines
    // We need to find the block of lines that are most likely the MRZ
    
    if (mrzLines.length < 2) {
      error.value = 'Could not detect MRZ in the document. Please ensure the MRZ area is clearly visible and crisp.'
      rawMrzText.value = text
      isProcessing.value = false
      return
    }
    
    // Attempt parsing
    // Strategy: 
    // 1. Try TD-3 (2 lines of 44 chars)
    // 2. Try TD-1 (3 lines of 30 chars)
    // 3. Use sliding window refined lines if available
    
    // We'll reuse the sliding window logic to get "clean" lines but target specific lengths
    
    let result: UnifiedMrzResult | null = null
    let errorMsg = ''
    
    // Helper to try parsing a block
    const tryParseCustom = (lines: string[]) => {
        try {
            if (lines.length === 2 && lines[0].length === 44 && lines[1].length === 44) {
                return parseTD3(lines)
            }
            if (lines.length === 3 && lines[0].length === 30 && lines[1].length === 30 && lines[2].length === 30) {
                return parseTD1(lines)
            }
        } catch (e) {
            // ignore
        }
        return null
    }
    
    // Generate candidates for 44 (Passport) and 30 (ID)
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

    // Try TD-3 (Last 2 lines, target 44)
    if (!result && mrzLines.length >= 2) {
        const last2 = mrzLines.slice(-2)
        const l1Candidates = getCandidates(last2[0], 44)
        const l2Candidates = getCandidates(last2[1], 44)
        
        for (const c1 of l1Candidates) {
            for (const c2 of l2Candidates) {
                const res = tryParseCustom([c1, c2])
                if (res && res.verification_status === 'VALID') {
                    result = res // prioritize valid
                    break
                }
                if (res) result = res // accept invalid if nothing better
            }
            if (result && result.verification_status === 'VALID') break
        }
    }

    // Try TD-1 (Last 3 lines, target 30)
    if ((!result || result.verification_status === 'INVALID') && mrzLines.length >= 3) {
        const last3 = mrzLines.slice(-3)
        const l1C = getCandidates(last3[0], 30)
        const l2C = getCandidates(last3[1], 30)
        const l3C = getCandidates(last3[2], 30)

        for (const c1 of l1C) {
            for (const c2 of l2C) {
                for (const c3 of l3C) {
                   const res = tryParseCustom([c1, c2, c3])
                   if (res && res.verification_status === 'VALID') {
                       result = res
                       break
                   }
                   if (res && !result) result = res
                }
                if (result && result.verification_status === 'VALID') break
            }
            if (result && result.verification_status === 'VALID') break
        }
    }

    rawMrzText.value = mrzLines.join('\n')
    
    // Mapping to legacy state for UI continuity, but ideally we show the new structure
    // We'll update the 'parsedResult' to use the new Unified form and update the template separately?
    // Or just map it back to the interface expected by the template for now, and expose the new JSON.
    // The user wants "Желаемый формат вывода". I should store this in a new ref.

    if (result) {
        // We'll cast to any for the UI template to keep it working, 
        // OR update the template. Let's update the interface above later or mapped here.
        // The current 'parsedResult' interface is loose enough to just populate new fields 
        // if we change the type or add 'any'.
        
        parsedResult.value = {
            documentType: result.document_type === 'PASSPORT' ? 'Passport (TD-3)' : 'ID Card (TD-1)',
            documentNumber: result.document_number,
            firstName: result.names.given_names,
            lastName: result.names.surname,
            nationality: result.nationality,
            birthDate: result.birth_date,
            sex: result.sex,
            expirationDate: result.expiry_date,
            issuingCountry: result.nationality, // heuristic
            valid: result.verification_status === 'VALID',
            details: result.errors?.map(e => ({ label: 'Error', value: e })) || [],
            personal_number: result.personal_number,
            
            // New fields for specific display requested
            unifiedJson: result // Store the full object to display
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
