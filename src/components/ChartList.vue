<script setup lang="ts">
import {ref, onMounted} from 'vue'
import axios from 'axios'
import * as pako from 'pako'
import MonacoEditor from "@/components/MonacoEditor.vue";

interface Chart {
  name: string
  version: string
  appVersion: string
  type: string
  urls: string[]
  created: string
  description: string
}

const baseUrl = 'https://charts.prd.gtidev.net'

const charts = ref<Chart[]>([])
const selectedChart = ref<Chart | null>(null)
const valuesContent = ref<string>('')

const headers = [
  {title: 'Name', key: 'name', sortable: true},
  {title: 'Chart-Version', key: 'version', sortable: true},
  {title: 'App-Version', key: 'appVersion', sortable: true},
  {title: 'Description', key: 'description', sortable: false},
  {title: 'Created', key: 'created', sortable: true},
]

const fetchCharts = async () => {
  try {
    const response = await axios.get(`${baseUrl}/api/charts`)
    const chartsData = response.data
    charts.value = Object.keys(chartsData).map(chartName => {
      const chartVersions = chartsData[chartName]
      const latestVersion = chartVersions[0] // First version is typically the latest
      return {
        name: chartName,
        version: latestVersion?.version || 'N/A',
        appVersion: latestVersion?.appVersion || 'N/A',
        type: latestVersion?.type || 'N/A',
        created: latestVersion?.created ? new Date(latestVersion.created).toLocaleString() : 'N/A',
        urls: latestVersion?.urls,
        description: latestVersion?.description || 'No description available',
      }
    })
  } catch (err) {
    console.error('Error fetching charts:', err)
  }
}

const parseTar = (arrayBuffer: ArrayBuffer): Map<string, Uint8Array> => {
  const files = new Map<string, Uint8Array>()
  const view = new Uint8Array(arrayBuffer)
  let offset = 0

  while (offset < view.length) {
    // Read filename (offset 0, length 100)
    const nameBytes = view.slice(offset, offset + 100)
    const nameEnd = nameBytes.indexOf(0)
    const filename = new TextDecoder().decode(nameBytes.slice(0, nameEnd > 0 ? nameEnd : 100))

    if (!filename) break

    // Read file size (offset 124, length 12) - octal string
    const sizeBytes = view.slice(offset + 124, offset + 136)
    const sizeStr = new TextDecoder().decode(sizeBytes).trim().replace(/\0/g, '')
    const fileSize = parseInt(sizeStr, 8)

    if (isNaN(fileSize)) break

    // File data starts at offset 512
    const fileData = view.slice(offset + 512, offset + 512 + fileSize)
    files.set(filename, fileData)

    // Move to next file (512-byte aligned)
    offset += 512 + Math.ceil(fileSize / 512) * 512
  }

  return files
}

const downloadAndExtractChart = async (ev: MouseEvent, row: { item: Chart }) => {
  const chart = row.item

  selectedChart.value = chart
  valuesContent.value = ''

  try {
    console.log(chart)
    const url = `${baseUrl}/${chart.urls[0]}`
    const response = await axios.get(url, {
      responseType: 'arraybuffer'
    })

    const decompressed = pako.ungzip(new Uint8Array(response.data))
    const files = parseTar(decompressed.buffer)

    let valuesFile: Uint8Array | undefined
    for (const [filename, content] of files.entries()) {
      if (filename.endsWith('values.yaml') || filename.endsWith('values.yml')) {
        valuesFile = content
        break
      }
    }

    if (valuesFile) {
      valuesContent.value = new TextDecoder().decode(valuesFile)
    } else {
      console.error('values.yaml not found in the chart archive')
    }
  } catch (err) {
    console.error('Error downloading chart:', err)
  }
}

onMounted(() => {
  fetchCharts()
})
</script>

<template>
  <v-container fluid>
    <v-row>
      <v-col cols="7">
        <v-card>
          <v-card-title>Charts on {{ baseUrl }}</v-card-title>
          <v-card-text>
            <v-data-table :headers="headers" :items="charts" @click:row="downloadAndExtractChart" hide-default-footer>
              <template v-slot:item.name="{ item }">
                <span class="font-weight-bold">{{ item.name }}</span>
              </template>
              <template v-slot:item.description="{ item }">
                <span class="text-truncate" style="max-width: 300px; display: inline-block;">
                  {{ item.description }}
                </span>
              </template>
              <template v-slot:no-data>
                <v-alert type="info" class="ma-4">
                  No charts available
                </v-alert>
              </template>
            </v-data-table>
          </v-card-text>
        </v-card>
      </v-col>
      <v-col cols="5">
        <v-card>
          <v-card-title>{{ selectedChart?.name }}</v-card-title>
          <v-card-text v-if="valuesContent">
            <monaco-editor v-model="valuesContent" language="yaml" :height="800" read-only/>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<style scoped>
.text-truncate {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
</style>
