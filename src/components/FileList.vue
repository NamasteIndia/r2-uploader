<template>
  <form action="javascript:">
    <div class="font-bold italic">
      <div>File List</div>
    </div>

    <div class="mt-4 mb-4 flex items-center flex-wrap space-x-2">
      <button
        v-show="!selectMode"
        class="text-xs inline-block w-auto outline mb-0"
        style="padding: 0.3rem 0.5rem"
        @click="loadData"
        :aria-busy="loading"
        :disabled="loading || !endPoint"
      >
        Refresh
      </button>
      <button
        class="text-xs inline-block w-auto outline mb-0"
        style="padding: 0.3rem 0.5rem"
        @click="toggleSelectMode"
        :disabled="fileList.length === 0 || loading"
      >
        {{ selectMode && fileList.length ? 'Quit Selection Mode' : 'Selection Mode' }}
      </button>

      <div v-show="selectMode" class="w-full flex space-x-2 mt-2">
        <button
          :disabled="selectedFiles.length === 0"
          class="text-xs inline-block w-auto outline mb-0 border-red-500 text-red-500"
          style="padding: 0.3rem 0.5rem"
          @click="deleteSelectedFiles"
        >
          Delete Selected
        </button>
        <button
          :disabled="selectedFiles.length === 0"
          class="text-xs inline-block w-auto outline mb-0 border-blue-500 text-blue-500 dark:border-blue-400 dark:text-blue-400"
          style="padding: 0.3rem 0.5rem"
          @click="copySelectedFileUrls"
        >
          {{ copyButtonText }}
        </button>
      </div>
    </div>

    <div>
      <div class="text-xs" v-show="!loading && fileList.length === 0 && !loadDataErrorText">
        Seems like we got nothing here.
      </div>
      <div class="text-red-500 text-xs" v-show="loadDataErrorText">
        {{ loadDataErrorText }}
        <pre class="mt-2"><code class="text-xs">{{ loadDataErrorStack }}</code></pre>
      </div>
      <div class="text-xs mb-4" v-show="fileList.length > 0">
        <span class="font-bold">{{ globalCursor ? 'More than' : '' }}</span> {{ fileList.length }} file{{
          fileList.length === 1 ? '' : 's'
        }}, {{ parseByteSize(allFileSize) }} total.
        <div class="inline-flex space-x-2">
          <button
            class="inline outline px-2 py-1 text-xs w-auto mb-0"
            v-show="globalCursor"
            @click="loadData('more')"
            :aria-busy="loading"
            :disabled="loading"
          >
            Load next page
          </button>
        </div>
      </div>

      <div class="text-xs mb-2" v-show="fileList.length > 0">
        Sort by
        <select class="text-xs inline-block w-[10rem] mb-0" v-model="sort">
          <option value="0">Default</option>
          <option value="1">Date(newest first)</option>
          <option value="2">Date(oldest first)</option>
          <option value="3">Size(largest first)</option>
          <option value="4">Size(smallest first)</option>
        </select>
      </div>

      <div class="pb-4" v-show="fileList.length > 0">
        <label for="seeFolderStructure" class="text-xs" :aria-busy="reconstructing">
          <input
            type="checkbox"
            id="seeFolderStructure"
            v-model="seeFolderStructure"
            class="mr-2"
            :disabled="reconstructing"
          />
          Folder Structure
        </label>
      </div>

      <div>
        <div
          class="rounded-lg mb-2"
          :class="seeFolderStructure ? 'bg-neutral-50 dark:bg-[#333] p-2 shadow' : ''"
          v-for="folder in Object.keys(dirMap).map((el) => ({ name: el, timestamp: Date.now() }))"
          :key="folder.name + '_' + structureId"
        >
          <details open class="mb-0 pb-1">
            <summary class="text-xs" v-show="seeFolderStructure">
              {{ folder.name }}
            </summary>

            <div
              class="mb-2 text-xs"
              v-show="selectMode"
              @mouseenter="mouseOnSelectionCheckbox = true"
              @mouseleave="mouseOnSelectionCheckbox = false"
            >
              <label :for="folder.name">
                <input
                  name="select_all_for_folder"
                  class="mr-2"
                  type="checkbox"
                  :id="folder.name"
                  @change="handleFolderSelect(folder.name)"
                />
                Select All
              </label>
            </div>

            <!-- File List with size, type, date -->
            <div
              class="item mb-2 rounded text-sm py-1 flex items-center justify-between"
              :class="seeFolderStructure ? 'pl-4' : ''"
              v-for="item in dirMap[folder.name]"
              :key="item.key"
            >
              <div class="w-[2rem]" v-show="selectMode">
                <input
                  type="checkbox"
                  @change="updateSelectedFiles(item, folder.name)"
                  v-model="item.selected"
                  :id="item.key"
                />
              </div>

              <div
                class="name whitespace-nowrap text-left text-ellipsis overflow-hidden break-all flex items-center justify-between"
                style="width: calc(100% - 7rem)"
                :style="{
                  width: selectMode ? 'calc(100% - 2rem)' : 'calc(100% - 5rem)'
                }"
              >
                <div class="w-full overflow-hidden text-ellipsis whitespace-nowrap">
                  <a
                    :href="(customDomain ? customDomain : endPoint) + item.key"
                    target="_blank"
                    v-show="!selectMode"
                  >
                    {{ item.fileName }}
                  </a>
                  <label v-show="selectMode" :for="item.key" class="mb-0">
                    {{ item.fileName }}
                  </label>
                </div>

                <div class="ml-2 text-gray-500 text-xs whitespace-nowrap">
                  {{ parseByteSize(item.size) }} ·
                  {{ getFileType(item.fileName) }} ·
                  {{ formatDate(item.lastModified) }}
                </div>
              </div>

              <div class="actions w-[5rem] shrink-0 text-right" v-show="!selectMode">
                <button
                  style="border: none; padding: 0.2rem 0.3rem"
                  class="w-auto inline-block outline text-xs text-red-500 mb-0"
                  @click="deleteThisFile(item.key)"
                  :aria-busy="deletingKey === item.key"
                  :disabled="deletingKey === item.key"
                >
                  Delete
                </button>
              </div>
            </div>
          </details>
        </div>

        <div class="inline-flex space-x-2">
          <button
            class="inline outline px-2 py-1 text-xs w-auto mb-0"
            v-show="globalCursor"
            @click="loadData('more')"
            :aria-busy="loading"
            :disabled="loading"
          >
            Load next page
          </button>
        </div>
      </div>
    </div>
  </form>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'
import axios from 'axios'
import { useStatusStore } from '../store/status'
import { storeToRefs } from 'pinia'
import { nanoid } from 'nanoid'

let sort = ref('0')
let selectMode = ref(false)
let allFileSize = ref(0)
let fileList = ref([])
let selectedFiles = ref([])
let dirMap = ref({})
let seeFolderStructure = ref(true)
let reconstructing = ref(false)
let loading = ref(false)
let deletingKey = ref('')
let structureId = nanoid()
let mouseOnSelectionCheckbox = ref(false)
let copyButtonText = ref('Copy URLs')
let copyButtonDisabled = ref(false)
let globalCursor = ref('')
let loadDataErrorText = ref('')
let loadDataErrorStack = ref('')

let statusStore = useStatusStore()
let { uploading, endPointUpdated } = storeToRefs(statusStore)

let endPoint = localStorage.getItem('endPoint')
let apiKey = localStorage.getItem('apiKey')
let customDomain = localStorage.getItem('customDomain')

// Helper functions
function parseByteSize(size) {
  let units = ['B', 'KB', 'MB', 'GB', 'TB']
  let index = 0
  while (size > 1000) {
    size /= 1000
    index++
  }
  return `${size.toFixed(2)} ${units[index]}`
}

function getFileType(fileName) {
  const ext = fileName.split('.').pop()?.toLowerCase()
  if (!ext) return 'unknown'
  if (['jpg','jpeg','png','gif','webp'].includes(ext)) return 'image'
  if (['mp4','mkv','avi','mov'].includes(ext)) return 'video'
  if (['mp3','wav','flac'].includes(ext)) return 'audio'
  if (['zip','rar','7z','tar','gz'].includes(ext)) return 'archive'
  if (['apk','ipa'].includes(ext)) return 'apk'
  if (['pdf','doc','docx','txt'].includes(ext)) return 'document'
  return ext
}

function formatDate(dateStr) {
  if (!dateStr) return ''
  const d = new Date(dateStr)
  return d.toLocaleDateString(undefined, { year:'numeric', month:'short', day:'numeric' })
}

// --- other functions (toggleSelectMode, loadData, deleteThisFile, mapFilesToDir, etc.) remain same as your original code --- //

</script>
