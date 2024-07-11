<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import axios from 'axios'

// const baseURL = 'http://localhost:3000/api/'
const baseURL = 'https://vercel-scraper-4a1zfivo5-senthurans-projects.vercel.app/api'

const imageName = ref('')
const imageUrl = ref<string | null>(null)
const imageFile = ref<File | null>(null)
const fileInput = ref<any>(null)
const scrapedData = ref<any>(null)
const isLoading = ref(false)
const errorMessage = ref('')
const showModal = ref(false)
const returnType = ref('json') // Default to JSON
const selectedContentType = ref('')

const formattedJson = computed(() => {
  if (!scrapedData.value) return ''
  return returnType.value == 'json' ? formatJson(scrapedData.value) : scrapedData.value
})

const returnTypes = [
  {
    value: 'structured',
    label: 'JSON',
    icon: '🧱',
    description: 'Organized data, easy for further processing'
  },
  {
    value: 'readable',
    label: 'Easy to Read',
    icon: '📖',
    description: 'Formatted for human readability'
  }
]

const contentTypes = [
  {
    value: 'form',
    label: 'Form',
    icon: '📝',
    description: 'Structured data with labels and values'
  }
  // {
  //   value: 'table',
  //   label: 'Table',
  //   icon: '📊',
  //   description: 'Data organized in rows and columns'
  // },
  // {
  //   value: 'paragraph',
  //   label: 'Paragraph',
  //   icon: '📄',
  //   description: 'Unstructured text content'
  // }
]

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
    const response = await axios.get(`${baseURL}/health`)
    console.log('API Status:', response.data)
  } catch (error) {
    console.error('Error checking API status:', error)
  }
}

const scrapeData = async (selectedContentType: string, returnType: string) => {
  console.log(imageFile.value, selectedContentType, returnType)

  if (!imageFile.value || !selectedContentType || !returnType) {
    return
  }

  isLoading.value = true
  errorMessage.value = ''

  try {
    const base64Image = await fileToBase64(imageFile.value)

    const response = await axios.post<ScrapedData>(
      `${baseURL}/upload`,
      {
        image: base64Image,
        contentType: selectedContentType,
        returnType: returnType
      },
      {
        headers: {
          'Content-Type': 'application/json'
        }
      }
    )

    if (returnType == 'structured') {
      scrapedData.value = JSON.parse(response.data.data.content[0].text)
    } else {
      scrapedData.value = 'Genarating Human Readble for Text'

      const response2 = await axios.post<ScrapedData>(
        `${baseURL}/html-genarator`,
        {
          content: response.data.data.content[0].text
        },
        {
          headers: {
            'Content-Type': 'application/json'
          }
        }
      )

      scrapedData.value = response2.data.data.content[0].text
    }

    // scrapedData.value =
    //   returnType == 'json'
    //     ? JSON.parse(response.data.data.content[0].text)
    //     : response.data.data.content[0].text
    // console.log('Scraped data:', response.data)
  } catch (error) {
    console.error('Error scraping data:', error)
    errorMessage.value = 'Failed to scrape data. Please try again.'
  } finally {
    isLoading.value = false
  }
}

// const scrapeData = async () => {
//   if (!imageFile.value) {
//     errorMessage.value = 'No image file selected'
//     console.error('No image file selected')
//     return
//   }

//   isLoading.value = true
//   errorMessage.value = ''

//   try {
//     // Convert image to base64
//     const base64Image = await fileToBase64(imageFile.value)
//     // Send base64 image to API
//     const response = await axios.post<ScrapedData>(
//       'https://vercel-scraper-k6nl52uaa-senthurans-projects.vercel.app/api/upload',
//       {
//         image: base64Image
//       },
//       {
//         headers: {
//           'Content-Type': 'application/json'
//         }
//       }
//     )

//     console.log('Scraped data:', JSON.parse(response.data.data.content[0].text))
//     scrapedData.value = JSON.parse(response.data.data.content[0].text)
//     // Handle the scraped data here
//   } catch (error) {
//     console.error('Error scraping data:', error)
//     errorMessage.value = 'Failed to scrape data. Please try again.'
//   } finally {
//     isLoading.value = false
//   }
// }

const showScrapeOptions = () => {
  showModal.value = true
}

const closeModal = () => {
  showModal.value = false
  // selectedContentType.value = ''
  // returnType.value = 'json'
}

const confirmScrape = () => {
  console.log(selectedContentType.value, returnType.value)

  if (selectedContentType.value && returnType.value) {
    closeModal()
    scrapeData(selectedContentType.value, returnType.value)
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
              <span class="ml-2 font-bold text-white text-xl">Document Scraper</span>
            </a>
          </div>
        </div>
      </div>
    </nav>

    <!-- Main content area -->
    <div class="flex-1 flex flex-col md:flex-row">
      <!-- Left side: File upload and image display -->
      <div class="w-full md:w-1/2 p-6 md:p-8 flex flex-col">
        <h2 class="text-3xl md:text-3xl font-bold mb-6 text-gray-800">Upload Document</h2>
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
            @click="showScrapeOptions"
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
          <template v-if="returnType === 'structured'">
            <pre class="json-display text-sm md:text-base" v-html="formattedJson"></pre>
          </template>
          <template v-else>
            <div v-html="formattedJson"></div>
          </template>
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

    <!-- Scrape Options Modal -->
    <transition
      enter-active-class="ease-out duration-300"
      enter-from-class="opacity-0"
      enter-to-class="opacity-100"
      leave-active-class="ease-in duration-200"
      leave-from-class="opacity-100"
      leave-to-class="opacity-0"
    >
      <div
        v-if="showModal"
        class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50"
      >
        <div
          class="bg-white rounded-lg shadow-xl p-6 w-full max-w-md transform transition-all"
          :class="showModal ? 'scale-100 opacity-100' : 'scale-95 opacity-0'"
        >
          <h3 class="text-2xl font-bold mb-4 text-gray-800">Select Content Type</h3>
          <!-- Content Type Selection -->
          <p class="text-gray-600 mb-3">Choose the type of content you're trying to scrape:</p>
          <div class="space-y-3 mb-6">
            <button
              v-for="type in contentTypes"
              :key="type.value"
              @click="selectedContentType = type.value"
              class="w-full text-left p-4 rounded-lg border transition-colors duration-200 flex items-center"
              :class="
                selectedContentType === type.value
                  ? 'border-blue-500 bg-blue-50'
                  : 'border-gray-200 hover:border-blue-500 hover:bg-blue-50'
              "
            >
              <span class="text-xl mr-4" v-html="type.icon"></span>
              <div>
                <span class="font-semibold text-gray-800">{{ type.label }}</span>
                <p class="text-sm text-gray-600">{{ type.description }}</p>
              </div>
            </button>
          </div>

          <!-- Return Type Selection -->
          <p class="text-gray-600 mb-3">How would you like the results presented?</p>
          <div class="space-y-3 mb-6">
            <label
              v-for="type in returnTypes"
              :key="type.value"
              class="flex items-center p-3 rounded-lg border cursor-pointer transition-colors duration-200"
              :class="
                returnType === type.value
                  ? 'border-blue-500 bg-blue-50'
                  : 'border-gray-200 hover:border-blue-500 hover:bg-blue-50'
              "
            >
              <input
                type="radio"
                :value="type.value"
                v-model="returnType"
                class="form-radio text-blue-500 mr-3"
              />
              <div class="flex items-center">
                <!-- <span class="text-xl mr-3" v-html="type.icon"></span> -->
                <div>
                  <span class="font-semibold text-gray-800">{{ type.label }}</span>
                  <p class="text-sm text-gray-600">{{ type.description }}</p>
                </div>
              </div>
            </label>
          </div>

          <div class="mt-6 flex justify-end space-x-3">
            <button
              @click="closeModal"
              class="px-4 py-2 bg-gray-200 text-gray-800 rounded-lg hover:bg-gray-300 transition-colors duration-200"
            >
              Cancel
            </button>
            <button
              @click="confirmScrape"
              class="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition-colors duration-200"
              :disabled="!selectedContentType || !returnType"
            >
              Scrape
            </button>
          </div>
        </div>
      </div>
    </transition>
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
