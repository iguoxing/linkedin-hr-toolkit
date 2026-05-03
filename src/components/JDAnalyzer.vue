<template>
  <div>
    <h1 class="page-title">JD 关键词分析器</h1>
    <p class="page-desc">粘贴职位描述（JD），自动提取高频关键词并按重要度排序，辅助优化简历和 LinkedIn 资料。</p>

    <div class="card">
      <div class="card-title">粘贴职位描述</div>
      <textarea v-model="jdText" placeholder="将完整的 Job Description 粘贴在此处…&#10;支持英文/中文混排" style="min-height:180px" />
      <div style="margin-top:12px;display:flex;gap:10px;align-items:center;flex-wrap:wrap">
        <button class="btn btn-primary" @click="analyzeJD" :disabled="!jdText.trim()">分析 JD</button>
        <button class="btn btn-outline" @click="loadDemo">载入示例 JD</button>
        <button v-if="jdText" class="btn btn-sm btn-danger" @click="jdText='';result=null">清空</button>
        <span style="font-size:12px;color:var(--gray-400)" v-if="jdText">{{ jdText.length }} 字符</span>
      </div>
    </div>

    <template v-if="result">
      <!-- Score -->
      <div class="card">
        <div class="card-title">JD 概览</div>
        <div class="grid-2">
          <div>
            <div class="section-label">关键词数量</div>
            <div style="font-size:24px;font-weight:600;color:var(--blue)">{{ result.totalKeywords }}</div>
          </div>
          <div>
            <div class="section-label">高频词 TOP10</div>
            <div style="font-size:24px;font-weight:600;color:var(--blue)">{{ result.topWords.length }}</div>
          </div>
          <div>
            <div class="section-label">技术词</div>
            <div style="font-size:24px;font-weight:600;color:#1e40af">{{ result.techWords.length }}</div>
          </div>
          <div>
            <div class="section-label">软技能词</div>
            <div style="font-size:24px;font-weight:600;color:#166534">{{ result.softWords.length }}</div>
          </div>
        </div>
      </div>

      <!-- Top keywords with freq bar -->
      <div class="card">
        <div class="card-title">高频关键词（点击复制）</div>
        <div v-for="item in result.topWords" :key="item.word"
          style="display:flex;align-items:center;gap:10px;padding:6px 0;border-bottom:0.5px solid var(--gray-100);cursor:pointer"
          @click="copyKw(item.word)" title="点击复制">
          <div style="width:120px;font-size:13px;font-weight:500">{{ item.word }}</div>
          <div style="flex:1;height:8px;background:var(--gray-100);border-radius:4px;overflow:hidden">
            <div :style="`width:${item.pct}%;background:var(--blue);height:100%;border-radius:4px`"></div>
          </div>
          <div style="width:32px;text-align:right;font-size:12px;color:var(--gray-600)">×{{ item.count }}</div>
        </div>
        <span v-if="kwCopied" class="copy-success" style="margin-top:8px;display:inline-flex">✓ 已复制：{{ kwCopied }}</span>
      </div>

      <!-- Technical vs Soft -->
      <div class="grid-2" style="margin-bottom:0">
        <div class="card" style="margin-bottom:0">
          <div class="card-title">技术 / 硬技能关键词</div>
          <div class="tag-list">
            <span v-for="k in result.techWords" :key="k" class="kw-chip kw-med"
              @click="copyKw(k)" style="cursor:pointer">{{ k }}</span>
          </div>
          <p v-if="!result.techWords.length" style="font-size:12px;color:var(--gray-400)">未检测到明显技术词</p>
        </div>
        <div class="card" style="margin-bottom:0">
          <div class="card-title">软技能 / 通用能力词</div>
          <div class="tag-list">
            <span v-for="k in result.softWords" :key="k" class="kw-chip kw-high"
              @click="copyKw(k)" style="cursor:pointer">{{ k }}</span>
          </div>
          <p v-if="!result.softWords.length" style="font-size:12px;color:var(--gray-400)">未检测到明显软技能词</p>
        </div>
      </div>

      <!-- Resume match -->
      <div class="card" style="margin-top:16px">
        <div class="card-title">简历关键词对照</div>
        <p style="font-size:12px;color:var(--gray-600);margin-bottom:10px">粘贴你的简历文本，检查哪些 JD 关键词已覆盖、哪些缺失</p>
        <textarea v-model="resumeText" placeholder="粘贴简历正文…" style="min-height:100px" />
        <button class="btn btn-primary" style="margin-top:8px" @click="matchResume" :disabled="!resumeText.trim()">对照检查</button>

        <template v-if="matchResult">
          <div class="divider" />
          <div class="score-wrap">
            <div style="font-size:13px;font-weight:500">匹配度</div>
            <div class="score-bar">
              <div class="score-fill"
                :style="`width:${matchResult.score}%;background:${matchResult.score>=70?'#057642':matchResult.score>=40?'#ca8a04':'#dc2626'}`"></div>
            </div>
            <div class="score-num">{{ matchResult.score }}%</div>
          </div>
          <div class="grid-2">
            <div>
              <div class="section-label" style="color:var(--success)">已覆盖 ({{ matchResult.found.length }})</div>
              <div class="tag-list">
                <span v-for="k in matchResult.found" :key="k" class="kw-chip" style="background:#dcfce7;color:#166534;border:1px solid #86efac">{{ k }}</span>
              </div>
            </div>
            <div>
              <div class="section-label" style="color:#dc2626">缺失 ({{ matchResult.missing.length }})</div>
              <div class="tag-list">
                <span v-for="k in matchResult.missing" :key="k" class="kw-chip" style="background:#fee2e2;color:#991b1b;border:1px solid #fca5a5">{{ k }}</span>
              </div>
            </div>
          </div>
        </template>
      </div>
    </template>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const jdText = ref('')
const resumeText = ref('')
const result = ref(null)
const matchResult = ref(null)
const kwCopied = ref('')

const TECH_PATTERNS = [
  'python','javascript','java','sql','react','vue','node','docker','kubernetes','aws','gcp','azure',
  'machine learning','deep learning','tensorflow','pytorch','spark','hadoop','tableau','power bi',
  'figma','sketch','xd','photoshop','git','ci/cd','rest','graphql','typescript','go','rust','swift',
  'flutter','android','ios','devops','agile','scrum','jira','confluence','salesforce','hubspot',
  'excel','word','powerpoint','r语言','matlab','c++','c#','.net','spring','django','flask',
]
const SOFT_PATTERNS = [
  'communication','leadership','teamwork','collaboration','problem solving','analytical',
  'creativity','innovative','adaptable','detail-oriented','self-motivated','multitask',
  'interpersonal','presentation','negotiation','strategic','organized','responsible',
  '沟通','领导力','团队合作','协作','解决问题','分析','创新','自驱','细心','负责',
  '执行力','学习能力','抗压','独立思考','跨部门',
]
const STOPWORDS = new Set([
  'the','a','an','and','or','of','in','to','for','is','are','be','with','as','that','this',
  'we','you','our','will','have','has','experience','years','year','including','ability',
  'work','working','team','strong','good','excellent','please','must','required','preferred',
  'job','position','role','candidate','company','business','new','other','more','all',
  '的','了','和','在','是','有','我','你','他','她','它','们','这','那','个','一','不','也',
  '以','及','或','将','并','要','能','该','与','对','为','等','中','上','下','可','到',
])

function tokenize(text) {
  // split by non-word chars, keep CJK chars
  const tokens = text.toLowerCase().match(/[\w\u4e00-\u9fa5]+/g) || []
  return tokens.filter(t => t.length > 1 && !STOPWORDS.has(t) && !/^\d+$/.test(t))
}

function extractNgrams(text, n) {
  const words = text.toLowerCase().match(/[\w\u4e00-\u9fa5]+/g) || []
  const result = []
  for (let i = 0; i <= words.length - n; i++) {
    result.push(words.slice(i, i + n).join(' '))
  }
  return result
}

function analyzeJD() {
  const text = jdText.value
  const tokens = tokenize(text)
  const bigrams = extractNgrams(text, 2)
  const trigrams = extractNgrams(text, 3)

  // count
  const freq = {}
  tokens.forEach(t => { freq[t] = (freq[t] || 0) + 1 })
  bigrams.forEach(b => {
    const clean = b.replace(/\s+/g, ' ')
    if (TECH_PATTERNS.some(p => clean.includes(p)) || SOFT_PATTERNS.some(p => clean.includes(p))) {
      freq[b] = (freq[b] || 0) + 1
    }
  })
  trigrams.forEach(t => {
    if (TECH_PATTERNS.some(p => t.includes(p))) {
      freq[t] = (freq[t] || 0) + 1
    }
  })

  const sorted = Object.entries(freq).filter(([, c]) => c >= 2)
    .sort((a, b) => b[1] - a[1])
    .slice(0, 30)

  const maxCount = sorted[0]?.[1] || 1
  const topWords = sorted.slice(0, 15).map(([word, count]) => ({
    word, count, pct: Math.round(count / maxCount * 100)
  }))

  const lower = text.toLowerCase()
  const techWords = [...new Set(
    TECH_PATTERNS.filter(p => lower.includes(p)).map(p => capitalize(p))
  )].slice(0, 16)
  const softWords = [...new Set(
    SOFT_PATTERNS.filter(p => lower.includes(p)).map(p => capitalize(p))
  )].slice(0, 12)

  result.value = {
    totalKeywords: sorted.length,
    topWords,
    techWords,
    softWords,
    allKeywords: [...new Set([...techWords, ...softWords, ...topWords.map(t => t.word)])]
  }
  matchResult.value = null
}

function matchResume() {
  if (!result.value || !resumeText.value.trim()) return
  const rLower = resumeText.value.toLowerCase()
  const allKws = [...new Set([
    ...result.value.techWords,
    ...result.value.softWords,
    ...result.value.topWords.slice(0, 10).map(t => t.word),
  ])]
  const found = allKws.filter(k => rLower.includes(k.toLowerCase()))
  const missing = allKws.filter(k => !rLower.includes(k.toLowerCase()))
  const score = allKws.length ? Math.round(found.length / allKws.length * 100) : 0
  matchResult.value = { found, missing, score }
}

function capitalize(s) {
  return s.split(' ').map(w => w.charAt(0).toUpperCase() + w.slice(1)).join(' ')
}

async function copyKw(k) {
  try { await navigator.clipboard.writeText(k) } catch {}
  kwCopied.value = k
  setTimeout(() => (kwCopied.value = ''), 1800)
}

function loadDemo() {
  jdText.value = `We are looking for a Senior Product Manager to join our growing team.

Responsibilities:
- Define product vision and roadmap for our SaaS platform
- Work closely with engineering, design, and marketing teams
- Write clear PRDs and user stories for feature development
- Analyze user data using SQL and analytics tools (Tableau, Mixpanel)
- Run A/B testing experiments to improve conversion rates
- Define KPIs and track product metrics via dashboards
- Manage the full product lifecycle from ideation to launch

Requirements:
- 5+ years of product management experience in a B2B SaaS company
- Strong analytical skills and data-driven decision making
- Experience with Agile/Scrum development methodology
- Excellent communication and stakeholder management skills
- Proficiency in SQL for data analysis; experience with Python is a plus
- Experience with project management tools such as Jira and Confluence
- Strong problem-solving skills and attention to detail
- Proven leadership ability and cross-functional collaboration experience
- MBA or equivalent experience preferred`
}
</script>
