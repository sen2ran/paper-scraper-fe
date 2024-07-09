<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import axios from 'axios'

const imageName = ref('')
const imageUrl = ref<string | null>(null)
const imageFile = ref<File | null>(null)
const fileInput = ref<any>(null)
const scrapedData = ref<any>(null)
const isLoading = ref(false)
const errorMessage = ref('')

const onFileSelected = (event: Event) => {
  const target = event.target as HTMLInputElement
  const file = target.files?.[0]
  if (file) {
    imageFile.value = file
    imageUrl.value = URL.createObjectURL(file)
    imageName.value = file.name
  }
}

const checkApiStatus = async () => {
  try {
    const response = await axios.get(
      'https://vercel-scraper-k6nl52uaa-senthurans-projects.vercel.app/api/health'
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
      'https://vercel-scraper-k6nl52uaa-senthurans-projects.vercel.app/api/upload',
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

const removeImage = () => {
  imageUrl.value = null
  imageFile.value = null
  imageName.value = ''
  if (fileInput.value) {
    fileInput.value.value = ''
  }
}

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
  <div class="flex flex-col min-h-screen bg-gray-50">
    <!-- Navigation bar -->
    <nav class="bg-gradient-to-r from-blue-600 to-indigo-600 shadow-lg">
      <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div class="flex items-center justify-between h-16">
          <div class="flex items-center">
            <a href="#" class="flex items-center">
              <svg class="h-8 w-8 text-white" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"
                />
              </svg>
              <span class="ml-2 font-bold text-white text-xl">Docu Scraper</span>
            </a>
          </div>
        </div>
      </div>
    </nav>

    <!-- Main content area -->
    <div class="flex-1 flex flex-col md:flex-row">
      <!-- Left side: File upload and image display -->
      <div class="w-full md:w-1/2 p-6 md:p-8 flex flex-col">
        <h2 class="text-2xl md:text-3xl font-bold mb-6 text-gray-800">Upload Document</h2>
        <div class="mb-6 flex flex-wrap gap-4">
          <input
            type="file"
            @change="onFileSelected"
            accept="image/*"
            ref="fileInput"
            class="hidden"
          />
          <button
            @click="fileInput.click()"
            class="bg-blue-500 hover:bg-blue-600 text-white font-semibold py-2 px-6 rounded-full shadow-md transition duration-300 ease-in-out transform hover:scale-105"
          >
            Select Image
          </button>
          <button
            @click="scrapeData"
            class="bg-green-500 hover:bg-green-600 text-white font-semibold py-2 px-6 rounded-full shadow-md transition duration-300 ease-in-out transform hover:scale-105"
            :disabled="!imageUrl"
            :class="{ 'opacity-50 cursor-not-allowed': !imageUrl }"
          >
            Scrape Data
          </button>
        </div>
        <div
          v-if="imageUrl"
          class="flex-1 flex flex-col items-center justify-center bg-gray-100 rounded-lg shadow-inner p-4 overflow-hidden"
        >
          <div
            class="relative w-full max-w-2xl aspect-[3/4] bg-white rounded-lg shadow-md overflow-hidden"
          >
            <img
              :src="imageUrl"
              alt="Uploaded document"
              class="absolute inset-0 w-full h-full object-contain"
            />
          </div>
          <p class="mt-4 text-sm text-gray-600">{{ imageName }}</p>
          <button @click="removeImage" class="mt-2 text-red-500 hover:text-red-600 font-semibold">
            Remove Image
          </button>
        </div>
        <div
          v-else
          class="flex-1 flex items-center justify-center bg-gray-100 rounded-lg border-2 border-dashed border-gray-300 p-4"
        >
          <p class="text-gray-500 text-center">
            No image uploaded. <br />
            Select an image to display it here.
          </p>
        </div>
      </div>
      <!-- Right side: Extracted data -->
      <div class="w-full md:w-1/2 p-6 md:p-8 bg-white shadow-md">
        <h2 class="text-2xl md:text-3xl font-bold mb-6 text-gray-800">Extracted Data</h2>
        <div
          v-if="scrapedData"
          class="bg-gray-100 rounded-lg p-4 overflow-auto max-h-[calc(100vh-12rem)]"
        >
          <pre class="json-display text-sm md:text-base" v-html="formattedJson"></pre>
        </div>
        <p v-else class="text-gray-600 italic">Scraped data will be displayed here.</p>
      </div>
    </div>

    <!-- Loader -->
    <div
      v-if="isLoading"
      class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50"
    >
      <div
        class="loader animate-spin rounded-full h-32 w-32 border-t-2 border-b-2 border-blue-500"
      ></div>
    </div>

    <!-- Error Popup -->
    <div
      v-if="errorMessage"
      class="fixed top-4 right-4 bg-red-500 text-white p-4 rounded-lg shadow-lg z-50 max-w-xs md:max-w-md animate-fade-in-down"
    >
      <p>{{ errorMessage }}</p>
      <button
        @click="errorMessage = ''"
        class="absolute top-2 right-2 text-white hover:text-gray-200"
      >
        <svg class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M6 18L18 6M6 6l12 12"
          />
        </svg>
      </button>
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

.animate-fade-in-down {
  animation: fadeInDown 0.5s ease-out;
}

@keyframes fadeInDown {
  0% {
    opacity: 0;
    transform: translateY(-20px);
  }
  100% {
    opacity: 1;
    transform: translateY(0);
  }
}

.json-display {
  white-space: pre-wrap;
  word-wrap: break-word;
}
</style>
