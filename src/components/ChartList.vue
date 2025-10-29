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

interface ChartWithVersions extends Chart {
  versions: Chart[]
}

interface ChangelogEntry {
  version: string
  changes: string[]
}

const baseUrl = import.meta.env.VITE_CHARTMUSEUM_URL

const charts = ref<ChartWithVersions[]>([])
const selectedChart = ref<Chart | null>(null)
const valuesContent = ref<string>('')
const changelogEntries = ref<ChangelogEntry[]>([])
const expanded = ref<string[]>([])

const headers = [
  {title: 'Name', key: 'name', sortable: true, width: 178},
  {title: 'Chart-Version', key: 'version', sortable: true, width: 200},
  {title: 'App-Version', key: 'appVersion', sortable: true, width: 140},
  {title: 'Description', key: 'description', sortable: false},
  {title: 'Created', key: 'created', sortable: true, width: 216},
] as const

const versionHeaders = [
  {title: '', key: 'indent', sortable: false, width: 178},
  {title: 'Chart-Version', key: 'version', sortable: false, width: 200},
  {title: 'App-Version', key: 'appVersion', sortable: false, width: 140},
  {title: 'Changelog', key: 'changelog', sortable: false},
  {title: 'Created', key: 'created', sortable: false, width: 200},
  {title: '', key: 'actions', sortable: false, width: 50, lign: 'end'},
] as const

const changelogHeaders = [
  {title: 'Version', key: 'version', sortable: false, width: 200},
  {title: 'Changes', key: 'changes', sortable: false},
] as const

const downloadChart = (chart: Chart) => {
  const url = `${baseUrl}/${chart.urls[0]}`
  const link = document.createElement('a')
  link.href = url
  link.download = `${chart.name}-${chart.version}.tgz`
  document.body.appendChild(link)
  link.click()
  document.body.removeChild(link)
}

const fetchCharts = async () => {
  try {
    const response = await axios.get(`${baseUrl}/api/charts`)
    const chartsData = response.data
    charts.value = Object.keys(chartsData).map(chartName => {
      const chartVersions = chartsData[chartName]
      const latestVersion = chartVersions[0] // First version is typically the latest

      // Map all versions
      const allVersions = chartVersions.map((v: any) => ({
        name: chartName,
        version: v.version || 'N/A',
        appVersion: v.appVersion || 'N/A',
        type: v.type || 'N/A',
        created: v.created ? new Date(v.created).toLocaleString() : 'N/A',
        urls: v.urls,
        description: v.description || 'No description available',
      }))

      return {
        name: chartName,
        version: latestVersion?.version || 'N/A',
        appVersion: latestVersion?.appVersion || 'N/A',
        type: latestVersion?.type || 'N/A',
        created: latestVersion?.created ? new Date(latestVersion.created).toLocaleString() : 'N/A',
        urls: latestVersion?.urls,
        description: latestVersion?.description || 'No description available',
        versions: allVersions,
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

const parseChangelog = (content: string): ChangelogEntry[] => {
  const entries: ChangelogEntry[] = []
  const lines = content.split('\n')

  let currentVersion: string | undefined = undefined
  let currentChanges: string[] = []

  for (const line of lines) {
    const versionMatch = line.match(/^###\s+(\d+\.\d+\.\d+)/)
    if (versionMatch) {
      // Save previous entry
      if (currentVersion) {
        entries.push({
          version: currentVersion,
          changes: currentChanges.filter(c => c.trim() !== '')
        })
      }
      // Start new entry
      currentVersion = versionMatch[1]
      currentChanges = []
    } else if (line.trim().startsWith('*') && currentVersion) {
      currentChanges.push(line.trim().substring(1).trim())
    }
  }

  // Save last entry
  if (currentVersion) {
    entries.push({
      version: currentVersion,
      changes: currentChanges.filter(c => c.trim() !== '')
    })
  }

  return entries.reverse()
}

const downloadAndExtractChart = async (chart: Chart, reloadChangelog: boolean) => {
  selectedChart.value = chart

  try {
    const url = `${baseUrl}/${chart.urls[0]}`
    const response = await axios.get(url, {
      responseType: 'arraybuffer'
    })

    const decompressed = pako.ungzip(new Uint8Array(response.data))
    const files = parseTar(decompressed.buffer)

    let valuesFile: Uint8Array | undefined
    let changelogFile: Uint8Array | undefined

    for (const [filename, content] of files.entries()) {
      if (filename.endsWith('values.yaml') || filename.endsWith('values.yml')) {
        valuesFile = content
      }
      if (filename.endsWith('CHANGELOG.md')) {
        changelogFile = content
      }
    }

    if (valuesFile) {
      valuesContent.value = new TextDecoder().decode(valuesFile)
    } else {
      console.error('values.yaml not found in the chart archive')
    }

    if (changelogFile && reloadChangelog) {
      const changelogContent = new TextDecoder().decode(changelogFile)
      changelogEntries.value = parseChangelog(changelogContent)
    } else {
      console.log('CHANGELOG.md not found in the chart archive')
    }
  } catch (err) {
    console.error('Error downloading chart:', err)
  }
}

const getChangelogForVersion = (version: string): string[] => {
  const entry = changelogEntries.value.find(e => e.version === version)
  if (entry && entry.changes.length > 0) {
    return entry.changes
  }
  return ['No changelog available']
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
          <v-card-title class="font-weight-light d-flex align-center">Charts on {{ baseUrl }}
            <v-spacer></v-spacer>
            <v-btn
              href="https://charts.prd.gtidev.net/api/charts" target="_blank"
              icon="mdi-open-in-new" size="small" variant="text" title="Open in ChartMuseum"
            />
          </v-card-title>
          <v-card-text>
            <v-data-table :headers="headers" :items="charts" v-model:expanded="expanded" item-value="name"
                          @click:row="(ev: MouseEvent, row: any) => downloadAndExtractChart(row.item,true)"
                          hide-default-footer show-expand>
              <template v-slot:expanded-row="{ columns, item }">
                <tr>
                  <td :colspan="columns.length" class="pa-0">
                    <v-data-table :headers="versionHeaders" :items="item.versions"
                                  @click:row="(ev: MouseEvent, row: any) => downloadAndExtractChart(row.item,false)"
                                  hide-default-footer hide-default-header density="compact">
                      <template v-slot:item.changelog="{ item }">
                        <ul class="my-2 pl-4">
                          <li v-for="(change, idx) in getChangelogForVersion(item.version)" :key="idx">{{
                              change
                            }}
                          </li>
                        </ul>
                      </template>
                      <template v-slot:item.actions="{ item }">
                        <v-btn icon="mdi-download" size="small" variant="text" @click.stop="downloadChart(item)"/>
                      </template>
                    </v-data-table>
                  </td>
                </tr>
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
          <v-card-title class="font-weight-light">
            Chart: {{ selectedChart?.name }} Version {{ selectedChart?.version }}
          </v-card-title>
        </v-card>
        <v-card class="mt-4">
          <v-card-title class="font-weight-light" style="font-size: large">Default Values</v-card-title>
          <v-card-text v-if="valuesContent">
            <monaco-editor v-model="valuesContent" language="yaml" :height="600" read-only/>
            <v-divider class="my-4" v-if="changelogEntries.length > 0"></v-divider>
          </v-card-text>
        </v-card>
        <v-card class="mt-4" v-if="changelogEntries.length > 0">
          <v-card-title class="font-weight-light" style="font-size: large">Changelog</v-card-title>
          <v-card-text>
            <v-data-table :headers="changelogHeaders" :items="changelogEntries" :items-per-page="-1"
                          hide-default-footer density="compact" height="300">
              <template v-slot:item.changes="{ item }">
                <ul class="my-2">
                  <li v-for="(change, idx) in item.changes" :key="idx">{{ change }}</li>
                </ul>
              </template>
            </v-data-table>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<style scoped>

</style>
