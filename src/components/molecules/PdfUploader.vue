<script setup lang="ts">
import { ref } from 'vue'
import Button from '@/components/atoms/Button.vue'
import * as pdfjsLib from 'pdfjs-dist'

// Configuration du worker PDF.js
pdfjsLib.GlobalWorkerOptions.workerSrc = `//cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjsLib.version}/pdf.worker.min.js`

const emit = defineEmits<{
  (e: 'extracted', data: {
    date?: string
    mileage?: string
    cost?: string
    garage?: string
    description?: string
  }): void
  (e: 'error', message: string): void
}>()

const processingPdf = ref(false)
const progress = ref(0)
const fileInput = ref<HTMLInputElement | null>(null)
const error = ref<string | null>(null)

const extractTextFromPdf = async (file: File): Promise<string> => {
  try {
    const arrayBuffer = await file.arrayBuffer()
    const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise
    let fullText = ''
    
    for (let i = 1; i <= pdf.numPages; i++) {
      progress.value = (i / pdf.numPages) * 100
      const page = await pdf.getPage(i)
      const textContent = await page.getTextContent()
      const pageText = textContent.items
        .map((item: any) => item.str)
        .join(' ')
      fullText += pageText + '\n'
    }
    
    return fullText
  } catch (err) {
    console.error('Erreur lors de l\'extraction du texte:', err)
    throw new Error('Impossible de lire le PDF. Vérifiez que le fichier n\'est pas corrompu.')
  }
}

const processPdf = async (file: File) => {
  try {
    error.value = null
    processingPdf.value = true
    progress.value = 0
    
    const text = await extractTextFromPdf(file)
    const extractedData: any = {}
    
    // Kilométrage (recherche de nombres suivis de "km")
    const mileageMatch = text.toLowerCase().match(/(\d+)\s*km/)
    if (mileageMatch) {
      extractedData.mileage = mileageMatch[1]
    }
    
    // Montant (recherche de nombres suivis de "€" ou "eur")
    const costMatch = text.toLowerCase().match(/(\d+[.,]\d{2})\s*(€|eur)/)
    if (costMatch) {
      extractedData.cost = costMatch[1].replace(',', '.')
    }
    
    // Date (format JJ/MM/AAAA ou JJ-MM-AAAA)
    const dateMatch = text.match(/(\d{2})[/-](\d{2})[/-](\d{4})/)
    if (dateMatch) {
      extractedData.date = `${dateMatch[3]}-${dateMatch[2]}-${dateMatch[1]}`
    }
    
    // Garage (recherche après "garage" ou au début d'une ligne)
    const garageMatch = text.toLowerCase().match(/garage\s+([^\n]+)/i)
    if (garageMatch) {
      extractedData.garage = garageMatch[1].trim()
    }
    
    // Description (recherche de mots clés courants d'entretien)
    const maintenanceKeywords = [
      'vidange', 'freins', 'pneus', 'révision',
      'courroie', 'distribution', 'filtre', 'batterie'
    ]
    
    const foundKeywords = maintenanceKeywords
      .filter(keyword => text.toLowerCase().includes(keyword))
      .join(', ')
    
    if (foundKeywords) {
      extractedData.description = `Intervention : ${foundKeywords}`
    }
    
    emit('extracted', extractedData)
  } catch (err: any) {
    const errorMessage = err.message || 'Une erreur est survenue lors du traitement du PDF'
    error.value = errorMessage
    emit('error', errorMessage)
  } finally {
    processingPdf.value = false
    progress.value = 0
  }
}

const handleFileSelect = async (event: Event) => {
  const input = event.target as HTMLInputElement
  if (input.files && input.files[0]) {
    const file = input.files[0]
    if (file.type === 'application/pdf') {
      await processPdf(file)
    } else {
      const errorMessage = 'Veuillez sélectionner un fichier PDF'
      error.value = errorMessage
      emit('error', errorMessage)
    }
  }
}
</script>

<template>
  <div>
    <Button
      variant="secondary"
      class="w-full"
      @click="fileInput?.click()"
      :disabled="processingPdf"
    >
      <div class="flex items-center justify-center space-x-2">
        <span v-if="!processingPdf">Analyser une facture PDF</span>
        <div v-else class="flex items-center space-x-2">
          <div class="animate-spin rounded-full h-4 w-4 border-b-2 border-black"></div>
          <span>Analyse en cours... {{ Math.round(progress) }}%</span>
        </div>
      </div>
    </Button>
    <input
      ref="fileInput"
      type="file"
      accept="application/pdf"
      class="hidden"
      @change="handleFileSelect"
    >
    <p class="mt-2 text-sm text-gray-500 text-center">
      Formats acceptés : PDF uniquement
    </p>
    <p v-if="error" class="mt-2 text-sm text-red-600 text-center">
      {{ error }}
    </p>
  </div>
</template>