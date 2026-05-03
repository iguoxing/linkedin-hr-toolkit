<template>
  <div>
    <h1 class="page-title">布尔搜索生成器</h1>
    <p class="page-desc">可视化拼接 AND / OR / NOT 搜索语句，一键复制到 LinkedIn 搜索框使用。</p>

    <div class="card">
      <div class="card-title">添加关键词</div>
      <div class="row">
        <div class="grow">
          <label>关键词 / 短语</label>
          <input type="text" v-model="inputWord" placeholder='例：product manager 或 "growth hacking"'
            @keyup.enter="addWord" />
        </div>
        <div>
          <label>类型</label>
          <select v-model="inputType" style="width:120px">
            <option value="must">必须包含 (AND)</option>
            <option value="should">任选其一 (OR)</option>
            <option value="not">排除 (NOT)</option>
          </select>
        </div>
        <div style="padding-top:22px">
          <button class="btn btn-primary" @click="addWord">+ 添加</button>
        </div>
      </div>

      <div class="divider" />

      <div v-if="mustWords.length || shouldWords.length || notWords.length">
        <div v-if="mustWords.length">
          <div class="section-label">必须包含 (AND)</div>
          <div class="op-group" v-if="mustWords.length>1">
            <button class="op-btn" :class="{'active-and': mustOp==='AND'}" @click="mustOp='AND'">AND</button>
            <button class="op-btn" :class="{'active-or': mustOp==='OR'}" @click="mustOp='OR'">OR</button>
          </div>
          <div class="tag-list">
            <span v-for="(w,i) in mustWords" :key="i" class="tag tag-must">
              {{ w }}
              <button class="tag-remove" @click="remove('must',i)">×</button>
            </span>
          </div>
        </div>

        <div v-if="shouldWords.length" style="margin-top:12px">
          <div class="section-label">任选其一 (OR)</div>
          <div class="tag-list">
            <span v-for="(w,i) in shouldWords" :key="i" class="tag tag-should">
              {{ w }}
              <button class="tag-remove" @click="remove('should',i)">×</button>
            </span>
          </div>
        </div>

        <div v-if="notWords.length" style="margin-top:12px">
          <div class="section-label">排除 (NOT)</div>
          <div class="tag-list">
            <span v-for="(w,i) in notWords" :key="i" class="tag tag-not">
              {{ w }}
              <button class="tag-remove" @click="remove('not',i)">×</button>
            </span>
          </div>
        </div>
      </div>
      <p v-else style="color:var(--gray-400);font-size:13px">暂无关键词，请在上方添加</p>
    </div>

    <div class="card">
      <div class="card-title">生成的搜索语句</div>
      <div class="output-box" :class="{'output-empty': !query}">
        {{ query || '添加关键词后自动生成…' }}
      </div>
      <div style="margin-top:12px;display:flex;align-items:center;gap:12px;flex-wrap:wrap">
        <button class="btn btn-primary" @click="copyQuery" :disabled="!query">复制语句</button>
        <button class="btn btn-outline" @click="openLinkedIn" :disabled="!query">
          在 LinkedIn 中搜索 ↗
        </button>
        <button class="btn btn-sm btn-danger" @click="clearAll" v-if="query">清空</button>
        <span v-if="copied" class="copy-success">✓ 已复制！</span>
      </div>
    </div>

    <div class="card">
      <div class="card-title">常用职位快速添加</div>
      <p style="font-size:12px;color:var(--gray-600);margin-bottom:8px">点击即添加为「必须包含」词</p>
      <div>
        <span v-for="s in suggestions" :key="s" class="suggest-badge" @click="quickAdd(s)">+ {{ s }}</span>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const inputWord = ref('')
const inputType = ref('must')
const mustWords = ref([])
const shouldWords = ref([])
const notWords = ref([])
const mustOp = ref('AND')
const copied = ref(false)

const suggestions = [
  'Product Manager', 'Software Engineer', 'Data Analyst', 'UX Designer',
  'Marketing Manager', 'Sales Manager', 'Growth Hacker', 'DevOps Engineer',
  'Machine Learning', 'Full Stack Developer', 'Project Manager', 'CTO'
]

function addWord() {
  const w = inputWord.value.trim()
  if (!w) return
  const quoted = w.includes(' ') && !w.startsWith('"') ? `"${w}"` : w
  if (inputType.value === 'must') mustWords.value.push(quoted)
  else if (inputType.value === 'should') shouldWords.value.push(quoted)
  else notWords.value.push(quoted)
  inputWord.value = ''
}

function quickAdd(s) {
  const quoted = s.includes(' ') ? `"${s}"` : s
  mustWords.value.push(quoted)
}

function remove(type, i) {
  if (type === 'must') mustWords.value.splice(i, 1)
  else if (type === 'should') shouldWords.value.splice(i, 1)
  else notWords.value.splice(i, 1)
}

const query = computed(() => {
  const parts = []
  if (mustWords.value.length) {
    parts.push(mustWords.value.join(` ${mustOp.value} `))
  }
  if (shouldWords.value.length) {
    const orPart = shouldWords.value.length > 1
      ? `(${shouldWords.value.join(' OR ')})`
      : shouldWords.value[0]
    parts.push(orPart)
  }
  let result = parts.join(' AND ')
  if (notWords.value.length) {
    result += ' NOT ' + notWords.value.join(' NOT ')
  }
  return result
})

async function copyQuery() {
  if (!query.value) return
  try {
    await navigator.clipboard.writeText(query.value)
    copied.value = true
    setTimeout(() => (copied.value = false), 2200)
  } catch {
    prompt('复制以下语句：', query.value)
  }
}

function openLinkedIn() {
  const url = `https://www.linkedin.com/search/results/all/?keywords=${encodeURIComponent(query.value)}`
  window.open(url, '_blank')
}

function clearAll() {
  mustWords.value = []
  shouldWords.value = []
  notWords.value = []
}

// Listen for imported query from KeywordExpander
function onImportQuery(e) {
  const q = e.detail
  if (!q) return
  shouldWords.value = []
  q.split(/\s+OR\s+/).forEach(word => {
    const clean = word.replace(/"/g, '').trim()
    if (clean) shouldWords.value.push(clean.includes(' ') ? `"${clean}"` : clean)
  })
}
onMounted(() => window.addEventListener('import-boolean-query', onImportQuery))
onUnmounted(() => window.removeEventListener('import-boolean-query', onImportQuery))
</script>
