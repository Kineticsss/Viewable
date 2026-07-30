<template>
  <div>
    <h1>SRT Timing Fixer</h1>
    <p class="sub">Shift subtitle timestamps forward or backward — no upload, runs entirely in your browser.</p>

    <!-- Drop zone -->
    <div
      v-if="!file"
      class="drop-zone"
      :class="{ over: isDragging }"
      @click="triggerInput"
      @dragover.prevent="isDragging = true"
      @dragleave="isDragging = false"
      @drop.prevent="onDrop"
    >
      <svg width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" style="display:block;margin:0 auto 12px"><path d="M4 16v2a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2v-2"/><polyline points="16 6 12 2 8 6"/><line x1="12" y1="2" x2="12" y2="15"/></svg>
      <p><strong>Drop your .srt file here</strong></p>
      <p style="margin-top:4px">or click to browse</p>
      <input ref="fileInput" type="file" accept=".srt" style="display:none" @change="onFileChange" />
    </div>

    <!-- Controls -->
    <div v-else class="card">
      <p class="card-title">Loaded file</p>

      <div class="file-row">
        <span class="file-name">{{ file.name }}</span>
        <span class="badge">{{ subtitleCount }} subtitles</span>
      </div>

      <div class="offset-row">
        <label for="offset">Offset (ms)</label>
        <input id="offset" type="number" v-model.number="offsetMs" step="50" />
      </div>
      <p class="offset-hint">
        <template v-if="offsetMs > 0">Shifts subtitles {{ offsetMs }}ms later ▶</template>
        <template v-else-if="offsetMs < 0">Shifts subtitles {{ Math.abs(offsetMs) }}ms earlier ◀</template>
        <template v-else>+ to shift later &nbsp;− to shift earlier</template>
      </p>

      <div class="preview-box" v-if="preview.before">
        <div><span class="label">before </span><span class="before">{{ preview.before }}</span></div>
        <div><span class="label">after &nbsp;&nbsp;</span><span class="after">{{ preview.after }}</span></div>
      </div>

      <div class="actions">
        <button class="btn-primary" :disabled="offsetMs === 0" @click="download">
          <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" y1="15" x2="12" y2="3"/></svg>
          Download fixed .srt
        </button>
        <button class="btn-ghost" @click="reset">
          <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="1 4 1 10 7 10"/><path d="M3.51 15a9 9 0 1 0 .49-3.45"/></svg>
          Load another file
        </button>
      </div>

      <p class="status" :class="statusClass">{{ statusMsg }}</p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const file = ref(null)
const rawContent = ref('')
const offsetMs = ref(0)
const isDragging = ref(false)
const statusMsg = ref('')
const statusClass = ref('')
const fileInput = ref(null)

const subtitleCount = computed(() => {
  const matches = rawContent.value.match(/^\d+$/gm)
  return matches ? matches.length : 0
})

function toMs(t) {
  const [h, m, sm] = t.split(':')
  const [s, ms] = sm.split(',')
  return +h * 3600000 + +m * 60000 + +s * 1000 + +ms
}

function toTime(ms) {
  ms = Math.max(0, ms)
  const h = Math.floor(ms / 3600000); ms %= 3600000
  const m = Math.floor(ms / 60000);   ms %= 60000
  const s = Math.floor(ms / 1000);    ms %= 1000
  return `${String(h).padStart(2,'0')}:${String(m).padStart(2,'0')}:${String(s).padStart(2,'0')},${String(ms).padStart(3,'0')}`
}

function shiftContent(content, offset) {
  return content.replace(
    /(\d{2}:\d{2}:\d{2},\d{3}) --> (\d{2}:\d{2}:\d{2},\d{3})/g,
    (_, a, b) => `${toTime(toMs(a) + offset)} --> ${toTime(toMs(b) + offset)}`
  )
}

const preview = computed(() => {
  const match = rawContent.value.match(/(\d{2}:\d{2}:\d{2},\d{3} --> \d{2}:\d{2}:\d{2},\d{3})/)
  if (!match) return {}
  const before = match[1]
  const after = shiftContent(before, offsetMs.value)
  return { before, after }
})

function loadFile(f) {
  if (!f || !f.name.endsWith('.srt')) {
    setStatus('Please upload a valid .srt file.', 'err')
    return
  }
  const reader = new FileReader()
  reader.onload = e => {
    file.value = f
    rawContent.value = e.target.result
    offsetMs.value = 0
    setStatus('', '')
  }
  reader.readAsText(f, 'utf-8')
}

function triggerInput() { fileInput.value?.click() }
function onFileChange(e) { loadFile(e.target.files[0]) }
function onDrop(e) { isDragging.value = false; loadFile(e.dataTransfer.files[0]) }

function download() {
  const fixed = shiftContent(rawContent.value, offsetMs.value)
  const blob = new Blob([fixed], { type: 'text/plain' })
  const a = document.createElement('a')
  a.href = URL.createObjectURL(blob)
  a.download = file.value.name.replace('.srt', '_fixed.srt')
  a.click()
  setStatus('Downloaded!', 'ok')
}

function reset() {
  file.value = null
  rawContent.value = ''
  offsetMs.value = 0
  setStatus('', '')
  if (fileInput.value) fileInput.value.value = ''
}

function setStatus(msg, cls) {
  statusMsg.value = msg
  statusClass.value = cls
}
</script>
