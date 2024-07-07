<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import axios from 'axios'

const imageUrl = ref<string | null>(null)
const imageFile = ref<File | null>(null)
const fileInput = ref<HTMLInputElement | null>(null)
const scrapedData = ref<any>(null)
const isLoading = ref(false)
const errorMessage = ref('')

const onFileSelected = (event: Event) => {
  const target = event.target as HTMLInputElement
  const file = target.files?.[0]
  if (file) {
    imageFile.value = file
    imageUrl.value = URL.createObjectURL(file)
  }
}

const checkApiStatus = async () => {
  try {
    const response = await axios.get(
      'https://vercel-scraper-n6c0adiwj-senthurans-projects.vercel.app/api/health'
    )
    console.log('API Status:', response.data)
  } catch (error) {
    console.error('Error checking API status:', error)
  }
}

const scrapeData = async () => {
  if (!imageFile.value) {
    errorMessage.value = 'No image file selected'
    console.error('No image file selected')
    return
  }

  isLoading.value = true
  errorMessage.value = ''

  try {
    // Convert image to base64
    const base64Image = await fileToBase64(imageFile.value)

    // Send base64 image to API
    const response = await axios.post<ScrapedData>(
      'https://vercel-scraper-n6c0adiwj-senthurans-projects.vercel.app/api/upload',
      {
        image: base64Image
      },
      {
        headers: {
          'Content-Type': 'application/json'
        }
      }
    )

    console.log('Scraped data:', JSON.parse(response.data.data.content[0].text))
    scrapedData.value = JSON.parse(response.data.data.content[0].text)
    // Handle the scraped data here
  } catch (error) {
    console.error('Error scraping data:', error)
    errorMessage.value = 'Failed to scrape data. Please try again.'
  } finally {
    isLoading.value = false
  }
}

const fileToBase64 = (file: File): Promise<string> => {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.readAsDataURL(file)
    reader.onload = () => {
      const result = reader.result
      if (typeof result === 'string') {
        resolve(result.split(',')[1])
      } else {
        reject(new Error('Failed to convert file to base64'))
      }
    }
    reader.onerror = (error) => reject(error)
  })
}

const formattedJson = computed(() => {
  if (!scrapedData.value) return ''
  return formatJson(scrapedData.value)
})

function formatJson(obj: any): string {
  return JSON.stringify(obj, null, 2)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(
      // eslint-disable-next-line no-useless-escape
      /("(\\u[a-zA-Z0-9]{4}|\\[^u]|[^\\"])*"(\s*:)?|\b(true|false|null)\b|-?\d+(?:\.\d*)?(?:[eE][+\-]?\d+)?)/g,
      (match) => {
        let cls = 'number'
        if (/^"/.test(match)) {
          if (/:$/.test(match)) {
            cls = 'key'
          } else {
            cls = 'string'
          }
        } else if (/true|false/.test(match)) {
          cls = 'boolean'
        } else if (/null/.test(match)) {
          cls = 'null'
        }
        return `<span class="${cls}">${match}</span>`
      }
    )
}

// Define an interface for the scraped data structure
// Update this to match the actual structure of your API response

onMounted(() => {
  checkApiStatus()
})

interface ScrapedData {
  message: string
  data: {
    [key: string]: any
  }
}
</script>

<template>
  <div class="flex flex-col h-screen bg-gray-100">
    <!-- Navigation bar -->
    <nav class="bg-blue-600 shadow-lg">
      <div class="max-w-6xl mx-auto px-4">
        <div class="flex justify-between">
          <div class="flex space-x-7">
            <div>
              <a href="#" class="flex items-center py-4 px-2">
                <span class="font-semibold text-white text-lg">Document Scraper</span>
              </a>
            </div>
          </div>
        </div>
      </div>
    </nav>

    <!-- Main content area -->
    <div class="flex flex-1 overflow-hidden">
      <!-- Left side: File upload and image display -->
      <div class="w-1/2 p-6 flex flex-col overflow-auto">
        <h2 class="text-2xl font-bold mb-4">Upload Document</h2>
        <div class="mb-4">
          <input
            type="file"
            @change="onFileSelected"
            accept="image/*"
            ref="fileInput"
            class="hidden"
          />
          <button
            @click="fileInput.click()"
            class="bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded mr-2"
          >
            Select Image
          </button>
          <button
            @click="scrapeData"
            class="bg-green-500 hover:bg-green-600 text-white font-bold py-2 px-4 rounded"
            :disabled="!imageUrl"
          >
            Scrape Data
          </button>
        </div>
        <div v-if="imageUrl" class="flex-1 overflow-auto">
          <img :src="imageUrl" alt="Uploaded document" class="max-w-full h-auto object-contain" />
        </div>
      </div>

      <!-- Right side: Extracted data -->
      <div class="w-1/2 p-6 bg-white overflow-auto">
        <h2 class="text-2xl font-bold mb-4">Extracted Data</h2>
        <pre v-if="scrapedData" class="json-display" v-html="formattedJson"></pre>
        <p v-else class="text-gray-600">Scraped data will be displayed here.</p>
      </div>
    </div>

    <!-- Loader -->
    <div
      v-if="isLoading"
      class="absolute inset-0 bg-black bg-opacity-50 flex items-center justify-center"
    >
      <div class="loader"></div>
    </div>

    <!-- Error Popup -->
    <div
      v-if="errorMessage"
      class="absolute top-4 right-4 bg-red-500 text-white p-4 rounded shadow-lg"
    >
      {{ errorMessage }}
      <button @click="errorMessage = ''" class="ml-4 font-bold">×</button>
    </div>
  </div>
</template>

<style scoped>
.json-display {
  font-family: monospace;
  font-size: 14px;
  line-height: 1.5;
  background-color: #f5f5f5;
  padding: 1rem;
  border-radius: 4px;
  white-space: pre-wrap;
  word-wrap: break-word;
}

.json-display .string {
  color: #008000;
}
.json-display .number {
  color: #0000ff;
}
.json-display .boolean {
  color: #b22222;
}
.json-display .null {
  color: #808080;
}
.json-display .key {
  color: #a52a2a;
}

.loader {
  border: 5px solid #f3f3f3;
  border-top: 5px solid #3498db;
  border-radius: 50%;
  width: 50px;
  height: 50px;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>
