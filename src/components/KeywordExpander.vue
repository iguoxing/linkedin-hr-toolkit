<template>
  <div>
    <h1 class="page-title">关键词拓词器</h1>
    <p class="page-desc">输入一个种子关键词，通过 Datamuse API 实时获取同义词、相关词、触发词，智能扩展你的 LinkedIn 搜索词库。</p>

    <div class="card">
      <div class="row">
        <div class="grow">
          <label>种子关键词</label>
          <input type="text" v-model="seed" placeholder="例：management, javascript, marketing"
            @keyup.enter="expand" />
        </div>
        <div style="padding-top:22px">
          <button class="btn btn-primary" @click="expand" :disabled="loading || !seed.trim()">
            {{ loading ? '拓词中…' : '开始拓词' }}
          </button>
        </div>
      </div>
      <div style="margin-top:10px">
        <span v-for="s in quickSeeds" :key="s" class="suggest-badge" @click="seed=s;expand()">{{ s }}</span>
      </div>
    </div>

    <template v-if="results">
      <!-- Summary -->
      <div class="card">
        <div class="card-title">拓词结果：{{ seed }}</div>
        <div style="display:grid;grid-template-columns:repeat(auto-fit,minmax(110px,1fr));gap:10px">
          <div class="metric-card" v-for="g in groups" :key="g.key">
            <div class="metric-label">{{ g.label }}</div>
            <div class="metric-value" :style="{color: g.color}">{{ g.words.length }}</div>
          </div>
        </div>
        <p style="font-size:12px;color:var(--gray-400);margin-top:12px;text-align:right">
          点击任意词可复制，双击可进一步拓词
        </p>
      </div>

      <!-- Word Groups -->
      <div v-for="g in groups" :key="'g-'+g.key" class="card">
        <div class="card-title">{{ g.label }}
          <span style="font-weight:400;font-size:12px;color:var(--gray-400);margin-left:6px">({{ g.words.length }})</span>
        </div>
        <div class="tag-list" v-if="g.words.length">
          <span v-for="w in g.words" :key="w.word"
            class="kw-chip" :class="g.chipClass"
            @click="copyWord(w.word)" style="cursor:pointer"
            @dblclick="seed=w.word;expand()"
            :title="'点击复制 / 双击进一步拓词\nscore: ' + w.score">
            {{ w.word }}
          </span>
        </div>
        <p v-else style="font-size:12px;color:var(--gray-400)">无结果</p>
      </div>

      <!-- Export -->
      <div class="card">
        <div class="card-title">导出为布尔搜索语句</div>
        <p style="font-size:12px;color:var(--gray-600);margin-bottom:8px">
          勾选要包含的词组，自动生成 OR 搜索语句
        </p>
        <div style="display:flex;flex-wrap:wrap;gap:10px;margin-bottom:12px">
          <label v-for="g in groups" :key="'chk-'+g.key" style="display:flex;align-items:center;gap:4px;font-size:12px;cursor:pointer;font-weight:400">
            <input type="checkbox" :value="g.key" v-model="exportGroups" style="accent-color:var(--blue)" />
            {{ g.label }}
          </label>
        </div>
        <div class="output-box" :class="{'output-empty': !exportQuery}">{{ exportQuery || '选择词组后自动生成…' }}</div>
        <div style="margin-top:10px;display:flex;gap:10px;align-items:center">
          <button class="btn btn-primary btn-sm" @click="copyExport" :disabled="!exportQuery">复制语句</button>
          <button class="btn btn-outline btn-sm" @click="sendToBoolean" :disabled="!exportQuery">发送到布尔搜索 →</button>
          <span v-if="exportCopied" class="copy-success">✓ 已复制</span>
        </div>
      </div>

      <!-- History -->
      <div class="card" v-if="history.length">
        <div class="card-title">拓词历史</div>
        <div class="tag-list">
          <span v-for="h in history" :key="h" class="suggest-badge" @click="seed=h;expand()">{{ h }}</span>
        </div>
      </div>
    </template>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const DATAMUSE = 'https://api.datamuse.com/words'
const seed = ref('')
const loading = ref(false)
const results = ref(null)
const history = ref([])
const exportGroups = ref(['syn', 'related', 'trigger'])
const exportCopied = ref(false)

const quickSeeds = ['management', 'javascript', 'marketing', 'design', 'leadership', 'analytics', 'strategy', 'engineer']

// API calls in parallel
async function expand() {
  if (!seed.value.trim() || loading.value) return
  loading.value = true
  const q = seed.value.trim().toLowerCase()
  try {
    const [synRes, relRes, trgRes] = await Promise.all([
      fetch(`${DATAMUSE}?rel_syn=${encodeURIComponent(q)}&max=20`).then(r => r.json()),
      fetch(`${DATAMUSE}?ml=${encodeURIComponent(q)}&max=20`).then(r => r.json()),
      fetch(`${DATAMUSE}?rel_trg=${encodeURIComponent(q)}&max=15`).then(r => r.json()),
    ])
    // Deduplicate: remove seed word itself
    const seen = new Set([q])
    const dedup = (arr) => arr.filter(w => w.word.toLowerCase() !== q && !seen.has(w.word.toLowerCase()) && !/[^a-zA-Z0-9\s\-\/.]/.test(w.word) && w.word.length > 2)
    const syn = dedup(synRes).slice(0, 15)
    syn.forEach(w => seen.add(w.word.toLowerCase()))
    const related = dedup(relRes).slice(0, 15)
    related.forEach(w => seen.add(w.word.toLowerCase()))
    const trigger = dedup(trgRes).slice(0, 15)
    results.value = { syn, related, trigger }
    if (!history.value.includes(q)) history.value.unshift(q)
  } catch (err) {
    alert('API 请求失败: ' + err.message)
  } finally {
    loading.value = false
  }
}

const groups = computed(() => {
  if (!results.value) return []
  return [
    { key: 'syn', label: '同义词 (Synonyms)', words: results.value.syn, color: '#1e40af', chipClass: 'kw-med' },
    { key: 'related', label: '语义相关词 (Related)', words: results.value.related, color: '#166534', chipClass: 'kw-high' },
    { key: 'trigger', label: '联想触发词 (Triggers)', words: results.value.trigger, color: '#92400e', chipClass: 'kw-low' },
  ]
})

const exportQuery = computed(() => {
  if (!results.value) return ''
  const words = []
  for (const key of exportGroups.value) {
    const g = groups.value.find(x => x.key === key)
    if (g) words.push(...g.words.map(w => w.word.includes(' ') ? `"${w.word}"` : w.word))
  }
  return words.length ? words.join(' OR ') : ''
})

async function copyWord(w) {
  try { await navigator.clipboard.writeText(w) } catch {}
}

async function copyExport() {
  try { await navigator.clipboard.writeText(exportQuery.value) } catch {}
  exportCopied.value = true
  setTimeout(() => exportCopied.value = false, 1800)
}

function sendToBoolean() {
  // Emit a custom event so BooleanSearch can pick it up
  window.dispatchEvent(new CustomEvent('import-boolean-query', { detail: exportQuery.value }))
  // Switch tab
  const navTabs = document.querySelectorAll('.nav-tab')
  navTabs[0]?.click()
}
</script>
