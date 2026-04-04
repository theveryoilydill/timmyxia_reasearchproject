<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref, computed } from 'vue'

type Source = {
  id: string
  name: string
  url: string
}

type Tab = {
  id: number
  title: string
  url: string
  origin: string
}

const allowedSources = ref<Source[]>([
  { id: 'poxel', name: 'Poxel (poxel.io)', url: 'https://poxel.io' },
  { id: 'gn-math', name: 'GN Math (gn-math.dev)', url: 'https://gn-math.dev' },
])

const tabs = ref<Tab[]>([])
const activeTabId = ref<number | null>(null)
const showAddDialog = ref(false)
const addSearchQuery = ref('')
const selectedSourceId = ref<string>(allowedSources.value[0]?.id || '')
const errorMessage = ref('')
let nextTabId = 1

const filteredSources = computed(() => {
  const q = addSearchQuery.value.trim().toLowerCase()
  if (!q) return allowedSources.value
  return allowedSources.value.filter((source) => source.name.toLowerCase().includes(q) || source.url.includes(q))
})

const isAllowedOrigin = (url: string) => {
  try {
    const parsed = new URL(url)
    return allowedSources.value.some((src) => new URL(src.url).origin === parsed.origin)
  } catch {
    return false
  }
}

const addTab = (source: Source) => {
  const origin = new URL(source.url).origin
  const exists = tabs.value.some((tab) => tab.url === source.url)
  if (exists) {
    activeTabId.value = tabs.value.find((tab) => tab.url === source.url)?.id ?? activeTabId.value
    return
  }

  const tab: Tab = {
    id: nextTabId++,
    title: source.name,
    url: source.url,
    origin,
  }
  tabs.value.push(tab)
  activeTabId.value = tab.id
}

const confirmAddTab = () => {
  errorMessage.value = ''
  const source = allowedSources.value.find((s) => s.id === selectedSourceId.value)
  if (!source) {
    errorMessage.value = 'No valid source selected.'
    return
  }

  if (!isAllowedOrigin(source.url)) {
    errorMessage.value = 'Selected source is not allowed.'
    return
  }

  addTab(source)
  closeAddDialog()
}

const closeAddDialog = () => {
  showAddDialog.value = false
  addSearchQuery.value = ''
  errorMessage.value = ''
}

const activateTab = (tabId: number) => {
  if (!tabs.value.some((tab) => tab.id === tabId)) return
  activeTabId.value = tabId
}

const closeTab = (tabId: number) => {
  const index = tabs.value.findIndex((tab) => tab.id === tabId)
  if (index === -1) return
  const removingActive = tabs.value[index].id === activeTabId.value
  tabs.value.splice(index, 1)

  if (removingActive) {
    if (tabs.value.length > 0) {
      const newIndex = Math.max(0, index - 1)
      activeTabId.value = tabs.value[newIndex].id
    } else {
      activeTabId.value = null
    }
  }
}

const openAddDialog = () => {
  showAddDialog.value = true
  selectedSourceId.value = filteredSources.value[0]?.id || allowedSources.value[0]?.id || ''
  addSearchQuery.value = ''
  errorMessage.value = ''
}

const handleKeydown = (e: KeyboardEvent) => {
  if ((e.altKey || e.metaKey) && e.key.toLowerCase() === 't') {
    e.preventDefault()
    openAddDialog()
    return
  }

  if (['ArrowRight', 'ArrowLeft', 'ArrowDown', 'ArrowUp'].includes(e.key)) {
    if (!showAddDialog.value) {
      e.preventDefault()
      openAddDialog()
    }
  }
}

onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeydown)
})
</script>

<template>
  <main class="home-view">
    <section class="top-controls">
      <button class="primary" type="button" @click="openAddDialog" title="Add tab (Alt+T)">
        + Add tab
      </button>
      <span class="hint">(Alt+T or Arrow keys to open add dialog)</span>

      <div class="tab-list" v-if="tabs.length > 0">
        <template v-for="tab in tabs" :key="tab.id">
          <button
            :class="['tab-item', { active: tab.id === activeTabId }]"
            @click="activateTab(tab.id)">
            {{ tab.title }}
            <span class="close" @click.stop="closeTab(tab.id)">×</span>
          </button>
        </template>
      </div>
    </section>

    <section class="pane">
      <div v-if="activeTabId === null" class="empty-state">
        Choose a tab or add one from poxel.io / gn-math.dev.
      </div>

      <div v-else class="iframe-wrapper">
        <iframe
          :src="tabs.find((t) => t.id === activeTabId)?.url"
          frameborder="0"
          sandbox="allow-scripts allow-same-origin"
          referrerpolicy="no-referrer"
          allow="clipboard-write; encrypted-media"
          title="Embedded tab"
        ></iframe>
      </div>
    </section>

    <div class="dialog-backdrop" v-if="showAddDialog" @click.self="closeAddDialog">
      <section class="add-dialog" role="dialog" aria-modal="true">
        <h2>Add allowed source tab</h2>
        <p>Only poxel.io and gn-math.dev are permitted, no redirects to external hosts.</p>

        <label class="search-field">
          Search sources:
          <input
            type="search"
            v-model="addSearchQuery"
            placeholder="Type to filter..."
            autofocus
          />
        </label>

        <label class="source-select">
          Source:
          <select v-model="selectedSourceId">
            <option
              v-for="source in filteredSources"
              :key="source.id"
              :value="source.id"
            >
              {{ source.name }}
            </option>
          </select>
        </label>

        <p v-if="errorMessage" class="error">{{ errorMessage }}</p>

        <div class="dialog-actions">
          <button class="primary" type="button" @click="confirmAddTab">Add tab</button>
          <button type="button" @click="closeAddDialog">Cancel</button>
        </div>
      </section>
    </div>
  </main>
</template>

<style scoped>
.home-view {
  display: flex;
  flex-direction: column;
  height: 100vh;
  padding: 0;
}

.top-controls {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 0.5rem;
  padding: 1rem;
}

.top-controls .hint {
  font-size: 0.85rem;
  color: #666;
}

.tab-list {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
}

.tab-item {
  border: 1px solid #ccc;
  background: #fff;
  border-radius: 0.5rem;
  padding: 0.3rem 0.6rem;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
}

.tab-item.active {
  border-color: #2563eb;
  background: #eff6ff;
}

.tab-item .close {
  margin-left: 0.5rem;
  font-weight: bold;
  cursor: pointer;
}

.pane {
  flex: 1;
  height: 100%;
  border: 1px solid #ddd;
  border-radius: 0.6rem;
  background: #fafafa;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #555;
}

.iframe-wrapper {
  width: 100%;
  height: 100%;
}

.iframe-wrapper iframe {
  width: 100%;
  height: 100%;
  border: 0;
}

.dialog-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.35);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 999;
}

.add-dialog {
  width: min(95vw, 420px);
  background: #fff;
  border-radius: 0.75rem;
  box-shadow: 0 12px 28px rgba(0, 0, 0, 0.25);
  padding: 1rem;
}

.add-dialog h2 {
  margin-top: 0;
}

.search-field,
.source-select {
  display: block;
  margin: 0.7rem 0;
}

.search-field input,
.source-select select {
  width: 100%;
  padding: 0.45rem;
  border: 1px solid #bbb;
  border-radius: 0.35rem;
}

.dialog-actions {
  display: flex;
  justify-content: flex-end;
  gap: 0.5rem;
  margin-top: 0.8rem;
}

.error {
  margin: 0.5rem 0;
  color: #b91c1c;
  font-weight: 600;
}

button.primary {
  background: #2563eb;
  color: white;
  border: 1px solid #2563eb;
  border-radius: 0.35rem;
  padding: 0.5rem 0.8rem;
  cursor: pointer;
}

button.primary:hover {
  background: #1e40af;
}

button:not(.primary) {
  border: 1px solid #999;
  background: #fff;
  border-radius: 0.35rem;
  padding: 0.5rem 0.8rem;
}
</style>
