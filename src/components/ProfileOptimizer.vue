<template>
  <div>
    <h1 class="page-title">个人资料关键词优化器</h1>
    <p class="page-desc">输入目标职位，获取 LinkedIn 个人资料应涵盖的高频关键词，提升搜索曝光。</p>

    <div class="card">
      <div class="card-title">目标职位</div>
      <div class="row">
        <div class="grow">
          <label>职位名称</label>
          <input type="text" v-model="targetRole" placeholder="例：产品经理、Data Scientist、Marketing Director" />
        </div>
        <div class="grow">
          <label>行业（可选）</label>
          <input type="text" v-model="industry" placeholder="例：互联网、金融科技、SaaS" />
        </div>
        <div style="padding-top:22px">
          <button class="btn btn-primary" @click="analyze">分析关键词</button>
        </div>
      </div>
    </div>

    <template v-if="analyzed">
      <!-- 匹配度 -->
      <div class="card">
        <div class="card-title">关键词建议</div>
        <p style="font-size:12px;color:var(--gray-600);margin-bottom:12px">点击关键词复制到剪贴板，填入 LinkedIn 标题、摘要或技能栏</p>

        <div class="section-label">高权重（必须出现在标题 / 摘要）</div>
        <div class="tag-list" style="margin-bottom:14px">
          <span v-for="k in result.high" :key="k" class="kw-chip kw-high"
            @click="copyKw(k)" style="cursor:pointer" title="点击复制">{{ k }}</span>
        </div>

        <div class="section-label">中权重（建议出现 2-3 次）</div>
        <div class="tag-list" style="margin-bottom:14px">
          <span v-for="k in result.mid" :key="k" class="kw-chip kw-med"
            @click="copyKw(k)" style="cursor:pointer" title="点击复制">{{ k }}</span>
        </div>

        <div class="section-label">补充关键词（扩大搜索覆盖）</div>
        <div class="tag-list">
          <span v-for="k in result.low" :key="k" class="kw-chip kw-low"
            @click="copyKw(k)" style="cursor:pointer" title="点击复制">{{ k }}</span>
        </div>

        <span v-if="kwCopied" class="copy-success" style="margin-top:10px;display:inline-flex">✓ 已复制：{{ kwCopied }}</span>
      </div>

      <!-- 资料各区域建议 -->
      <div class="card">
        <div class="card-title">各区域填写建议</div>
        <div class="grid-2">
          <div v-for="sec in sections" :key="sec.title">
            <div style="font-size:13px;font-weight:500;margin-bottom:4px">{{ sec.title }}</div>
            <div style="font-size:12px;color:var(--gray-600);line-height:1.7">{{ sec.tips }}</div>
          </div>
        </div>
      </div>

      <!-- 优化 Checklist -->
      <div class="card">
        <div class="card-title">优化 Checklist</div>
        <div v-for="(item, i) in checklist" :key="i"
          style="display:flex;align-items:flex-start;gap:10px;padding:7px 0;border-bottom:0.5px solid var(--gray-100)">
          <input type="checkbox" v-model="item.done" style="margin-top:3px;width:14px;height:14px;accent-color:var(--blue)" />
          <div>
            <div style="font-size:13px" :style="item.done?'text-decoration:line-through;color:var(--gray-400)':''">{{ item.text }}</div>
          </div>
        </div>
        <div style="margin-top:12px;font-size:13px;color:var(--gray-600)">
          完成度 {{ doneCount }} / {{ checklist.length }}
          <span v-if="doneCount===checklist.length" style="color:var(--success);font-weight:500;margin-left:8px">✓ 全部完成！</span>
        </div>
      </div>
    </template>
  </div>
</template>

<script setup>
import { ref, computed, reactive } from 'vue'

const targetRole = ref('')
const industry = ref('')
const analyzed = ref(false)
const kwCopied = ref('')
const result = reactive({ high: [], mid: [], low: [] })
const checklist = ref([])

// Keyword database by role pattern
const DB = {
  '产品经理|product manager|pm': {
    high: ['Product Manager', '产品经理', 'PRD', 'Roadmap', 'User Story', 'Agile', 'KPI'],
    mid: ['A/B Testing', 'OKR', 'Wireframe', 'Go-to-Market', 'Cross-functional', 'Stakeholder'],
    low: ['Jira', 'Figma', 'SQL', 'Data-driven', 'Sprint', 'MVP', 'Product Strategy'],
  },
  'data|数据': {
    high: ['Data Analysis', 'Python', 'SQL', 'Machine Learning', 'Data Science', 'Tableau'],
    mid: ['ETL', 'Power BI', 'Statistics', 'Excel', 'Data Visualization', 'Business Intelligence'],
    low: ['R', 'Spark', 'Hadoop', 'A/B Testing', 'Forecasting', 'Dashboard', 'Data Pipeline'],
  },
  'engineer|developer|工程师|开发': {
    high: ['Software Engineer', 'Full Stack', 'JavaScript', 'Python', 'React', 'Node.js'],
    mid: ['REST API', 'Git', 'CI/CD', 'Docker', 'Microservices', 'Cloud', 'AWS'],
    low: ['TypeScript', 'Kubernetes', 'GraphQL', 'Agile', 'TDD', 'DevOps', 'System Design'],
  },
  'market|营销|marketing': {
    high: ['Digital Marketing', 'SEO', 'SEM', 'Content Marketing', 'Growth', 'Brand'],
    mid: ['Google Ads', 'Social Media', 'Email Marketing', 'Analytics', 'Conversion', 'CRM'],
    low: ['HubSpot', 'Salesforce', 'ROI', 'Inbound', 'Lead Generation', 'Demand Generation'],
  },
  'design|ux|设计': {
    high: ['UX Design', 'UI Design', 'Figma', 'User Research', 'Prototyping', 'Interaction Design'],
    mid: ['Design System', 'Usability Testing', 'Wireframe', 'Sketch', 'Adobe XD', 'Design Thinking'],
    low: ['Adobe Illustrator', 'Accessibility', 'Visual Design', 'Information Architecture', 'Motion Design'],
  },
  'sales|销售': {
    high: ['Sales', 'B2B', 'Business Development', 'Pipeline', 'Quota', 'CRM'],
    mid: ['Salesforce', 'Cold Calling', 'Negotiation', 'Account Management', 'Lead Generation'],
    low: ['SaaS Sales', 'Revenue', 'Upselling', 'Outbound', 'Hunter', 'Channel Sales'],
  },
}

const DEFAULT = {
  high: ['Leadership', 'Project Management', 'Strategy', 'Communication', 'Team Collaboration'],
  mid: ['Problem Solving', 'Data-driven', 'Cross-functional', 'Stakeholder Management'],
  low: ['Microsoft Office', 'Presentation', 'Budget Management', 'Process Improvement'],
}

const sectionMap = {
  '标题 (Headline)': '将最重要的职位关键词写入标题，如：Product Manager | Growth | SaaS',
  '摘要 (About)': '前3行写核心价值主张，自然融入3-5个高权重关键词，以第一人称叙述',
  '工作经历': '每段经历用量化数据支撑，关键词要与职位描述保持一致',
  '技能栏': '至少填写10个技能，优先选高权重关键词，并获得他人背书',
  '项目 / 成就': '用 STAR 法则描述，嵌入中权重关键词',
  '推荐语': '请上级或同事在推荐语中自然提及核心能力关键词',
}

const CHECKLIST = [
  '标题包含目标职位头衔',
  '标题字符已接近 220 个字符上限',
  '摘要前 3 行包含核心关键词',
  '所有工作经历均包含量化成果',
  '技能栏 ≥ 10 个技能',
  '技能中至少 3 个获得 5+ 背书',
  '个人资料完整度达到 All-Star',
  '头像专业（无滤镜，背景简洁）',
  '背景图与职业方向相关',
  '已开启「向招聘者开放」状态',
]

function matchDB(role) {
  const lower = role.toLowerCase()
  for (const [pattern, kws] of Object.entries(DB)) {
    if (pattern.split('|').some(p => lower.includes(p))) return kws
  }
  return DEFAULT
}

function analyze() {
  if (!targetRole.value.trim()) return
  const kws = matchDB(targetRole.value)
  result.high = [...kws.high]
  result.mid = [...kws.mid]
  result.low = [...kws.low]
  if (industry.value) {
    result.high.unshift(industry.value)
  }
  checklist.value = CHECKLIST.map(text => reactive({ text, done: false }))
  analyzed.value = true
}

const sections = computed(() =>
  Object.entries(sectionMap).map(([title, tips]) => ({ title, tips }))
)

const doneCount = computed(() => checklist.value.filter(i => i.done).length)

async function copyKw(k) {
  try { await navigator.clipboard.writeText(k) } catch {}
  kwCopied.value = k
  setTimeout(() => (kwCopied.value = ''), 1800)
}
</script>
