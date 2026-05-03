<template>
  <div>
    <h1 class="page-title">实时职位探索</h1>
    <p class="page-desc">基于 Remotive API 实时获取远程职位数据，动态分析热门技能关键词趋势，辅助你的 LinkedIn 搜索策略。</p>

    <!-- Search -->
    <div class="card">
      <div class="row">
        <div class="grow">
          <label>搜索关键词</label>
          <input type="text" v-model="keyword" placeholder="例：software engineer, product manager, data"
            @keyup.enter="search" />
        </div>
        <div>
          <label>职位类别</label>
          <select v-model="category">
            <option value="">全部类别</option>
            <option v-for="c in categories" :key="c.slug" :value="c.slug">{{ c.name }}</option>
          </select>
        </div>
        <div style="padding-top:22px">
          <button class="btn btn-primary" @click="search" :disabled="loading">
            {{ loading ? '搜索中…' : '搜索职位' }}
          </button>
        </div>
      </div>
    </div>

    <template v-if="loaded">
      <!-- Stats -->
      <div class="card">
        <div class="card-title">数据概览</div>
        <div style="display:grid;grid-template-columns:repeat(auto-fit,minmax(120px,1fr));gap:12px">
          <div class="metric-card">
            <div class="metric-label">找到职位</div>
            <div class="metric-value">{{ totalJobs }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">分析数量</div>
            <div class="metric-value">{{ jobs.length }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">热门标签</div>
            <div class="metric-value">{{ topTags.length }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">涉及公司</div>
            <div class="metric-value">{{ uniqueCompanies }}</div>
          </div>
        </div>
      </div>

      <!-- Tag Cloud -->
      <div class="card">
        <div class="card-title">热门技能标签云</div>
        <p style="font-size:12px;color:var(--gray-600);margin-bottom:12px">
          基于 {{ jobs.length }} 条职位数据的 tags 字段统计，字号越大 = 出现频率越高。点击标签可快速搜索。
        </p>
        <div class="cloud-wrap" v-if="topTags.length">
          <span v-for="tag in topTags" :key="tag.name"
            class="cloud-tag"
            :style="{fontSize: tag.size + 'px', padding: tag.size/3 + 'px ' + tag.size/1.8 + 'px', opacity: 0.5 + tag.ratio * 0.5 }"
            @click="quickSearch(tag.name)"
            :title="tag.name + ': 出现 ' + tag.count + ' 次'">
            {{ tag.name }}
          </span>
        </div>
        <p v-else style="color:var(--gray-400);font-size:13px">暂无标签数据</p>
      </div>

      <!-- Tag Frequency Chart -->
      <div class="card">
        <div class="card-title">标签出现频率 TOP 15</div>
        <div v-for="tag in topTags.slice(0, 15)" :key="'bar-'+tag.name"
          style="display:flex;align-items:center;gap:8px;padding:4px 0">
          <div style="width:100px;font-size:12px;text-align:right;flex-shrink:0;white-space:nowrap;overflow:hidden;text-overflow:ellipsis" :title="tag.name">{{ tag.name }}</div>
          <div style="flex:1;height:6px;background:var(--gray-100);border-radius:3px;overflow:hidden">
            <div :style="`width:${tag.pct}%;height:100%;border-radius:3px;background:hsl(210,70%,${40+tag.ratio*30}%)`"></div>
          </div>
          <div style="width:28px;font-size:11px;color:var(--gray-600);text-align:right">{{ tag.count }}</div>
        </div>
      </div>

      <!-- Category Distribution -->
      <div class="card" v-if="categoryStats.length">
        <div class="card-title">职位类别分布</div>
        <div v-for="cat in categoryStats" :key="cat.name"
          style="display:flex;align-items:center;gap:8px;padding:4px 0">
          <div style="width:100px;font-size:12px;flex-shrink:0;white-space:nowrap;overflow:hidden;text-overflow:ellipsis" :title="cat.name">{{ cat.name }}</div>
          <div style="flex:1;height:6px;background:var(--gray-100);border-radius:3px;overflow:hidden">
            <div :style="`width:${cat.pct}%;height:100%;border-radius:3px;background:hsl(160,50%,${40+cat.ratio*25}%)`"></div>
          </div>
          <div style="width:40px;font-size:11px;color:var(--gray-600);text-align:right">{{ cat.count }}</div>
        </div>
      </div>

      <!-- Job Listings -->
      <div class="card">
        <div class="card-title">职位列表 (最新)</div>
        <div v-if="jobs.length">
          <div v-for="job in jobs" :key="job.id"
            class="job-item">
            <div style="display:flex;align-items:center;gap:10px;margin-bottom:6px">
              <img v-if="job.company_logo" :src="job.company_logo"
                style="width:36px;height:36px;border-radius:6px;object-fit:cover;background:var(--gray-100)"
                loading="lazy" @error="$event.target.style.display='none'" />
              <div style="flex:1;min-width:0">
                <div style="font-size:14px;font-weight:500;white-space:nowrap;overflow:hidden;text-overflow:ellipsis">
                  <a :href="job.url" target="_blank" rel="noopener" style="color:var(--blue);text-decoration:none">{{ job.title }}</a>
                </div>
                <div style="font-size:12px;color:var(--gray-600)">
                  {{ job.company_name }}
                  <span v-if="job.salary" style="margin-left:8px;color:var(--success)">· {{ job.salary }}</span>
                </div>
              </div>
              <span class="type-badge">{{ job.job_type }}</span>
            </div>
            <div class="tag-list" style="margin:0">
              <span v-for="tag in (job.tags||[]).slice(0,8)" :key="tag" class="kw-chip kw-med" style="font-size:11px;padding:1px 8px;cursor:pointer" @click="quickSearch(tag)">{{ tag }}</span>
            </div>
            <div style="font-size:11px;color:var(--gray-400);margin-top:4px">
              {{ job.candidate_required_location || 'Remote' }} · {{ formatDate(job.publication_date) }}
            </div>
          </div>
        </div>
        <div v-else style="color:var(--gray-400);font-size:13px">没有找到匹配的职位，试试其他关键词</div>
        <div style="text-align:center;margin-top:16px" v-if="jobs.length && jobs.length < totalJobs">
          <button class="btn btn-outline btn-sm" @click="loadMore" :disabled="loading">加载更多</button>
        </div>
      </div>
    </template>

    <!-- Initial state -->
    <div v-if="!loaded" class="card" style="text-align:center;padding:48px 24px">
      <svg width="48" height="48" viewBox="0 0 24 24" fill="var(--gray-200)" style="margin:0 auto 16px">
        <path d="M20.447 20.452h-3.554v-5.569c0-1.328-.027-3.037-1.852-3.037-1.853 0-2.136 1.445-2.136 2.939v5.667H9.351V9h3.414v1.561h.046c.477-.9 1.637-1.85 3.37-1.85 3.601 0 4.267 2.37 4.267 5.455v6.286zM5.337 7.433a2.062 2.062 0 01-2.063-2.065 2.064 2.064 0 112.063 2.065zm1.782 13.019H3.555V9h3.564v11.452zM22.225 0H1.771C.792 0 0 .774 0 1.729v20.542C0 23.227.792 24 1.771 24h20.451C23.2 24 24 23.227 24 22.271V1.729C24 .774 23.2 0 22.222 0h.003z"/>
      </svg>
      <p style="font-size:14px;color:var(--gray-600);margin-bottom:8px">输入关键词开始探索实时职位数据</p>
      <div style="display:flex;gap:6px;justify-content:center;flex-wrap:wrap">
        <span v-for="s in quickKeywords" :key="s" class="suggest-badge" @click="keyword=s;search()">{{ s }}</span>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

const API_BASE = 'https://remotive.com/api/remote-jobs'

const keyword = ref('')
const category = ref('')
const jobs = ref([])
const totalJobs = ref(0)
const loading = ref(false)
const loaded = ref(false)
const categories = ref([])

const quickKeywords = ['Software Engineer', 'Product Manager', 'Data Analyst', 'Designer', 'Marketing', 'DevOps']

onMounted(async () => {
  try {
    const res = await fetch(`${API_BASE}?limit=100`)
    const data = await res.json()
    if (data.jobs) {
      const catSet = new Map()
      data.jobs.forEach(j => {
        if (j.category && !catSet.has(j.category)) catSet.set(j.category, j.category)
      })
      categories.value = [...catSet.keys()].map(name => ({ slug: name, name }))
    }
  } catch {}
})

async function search() {
  if (loading.value) return
  loading.value = true
  try {
    const params = new URLSearchParams({ limit: '50' })
    if (keyword.value.trim()) params.set('search', keyword.value.trim())
    if (category.value) params.set('category', category.value)
    const res = await fetch(`${API_BASE}?${params}`)
    const data = await res.json()
    jobs.value = data.jobs || []
    totalJobs.value = data['job-count'] || data['total-job-count'] || jobs.value.length
    loaded.value = true
  } catch (err) {
    alert('请求失败: ' + err.message)
  } finally {
    loading.value = false
  }
}

async function loadMore() {
  if (loading.value) return
  loading.value = true
  try {
    const params = new URLSearchParams({ limit: '50', page: '2' })
    if (keyword.value.trim()) params.set('search', keyword.value.trim())
    if (category.value) params.set('category', category.value)
    const res = await fetch(`${API_BASE}?${params}`)
    const data = await res.json()
    const newJobs = data.jobs || []
    jobs.value = [...jobs.value, ...newJobs]
    totalJobs.value = data['job-count'] || data['total-job-count'] || jobs.value.length
  } catch {} finally {
    loading.value = false
  }
}

function quickSearch(tag) {
  keyword.value = tag
  search()
}

// Analyze tags frequency
const topTags = computed(() => {
  const freq = {}
  jobs.value.forEach(j => {
    (j.tags || []).forEach(t => {
      freq[t] = (freq[t] || 0) + 1
    })
  })
  const maxCount = Math.max(...Object.values(freq), 1)
  return Object.entries(freq)
    .map(([name, count]) => ({
      name,
      count,
      ratio: count / maxCount,
      pct: Math.round(count / maxCount * 100),
      size: Math.round(12 + (count / maxCount) * 22),
    }))
    .sort((a, b) => b.count - a.count)
})

// Category stats
const categoryStats = computed(() => {
  const freq = {}
  jobs.value.forEach(j => {
    const cat = j.category || 'Other'
    freq[cat] = (freq[cat] || 0) + 1
  })
  const maxCount = Math.max(...Object.values(freq), 1)
  return Object.entries(freq)
    .map(([name, count]) => ({
      name, count,
      ratio: count / maxCount,
      pct: Math.round(count / maxCount * 100),
    }))
    .sort((a, b) => b.count - a.count)
    .slice(0, 8)
})

const uniqueCompanies = computed(() => {
  return new Set(jobs.value.map(j => j.company_name)).size
})

function formatDate(d) {
  if (!d) return ''
  try { return new Date(d).toLocaleDateString('zh-CN') } catch { return d }
}
</script>
