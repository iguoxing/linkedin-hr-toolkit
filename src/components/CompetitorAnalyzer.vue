<template>
  <div>
    <h1 class="page-title">竞品 / 同行分析器</h1>
    <p class="page-desc">
      输入公司名称、LinkedIn 公司主页链接，或 LinkedIn 个人主页链接，
      自动获取公司资料、职位数据、技能画像，全方位分析人才策略与竞争格局。
    </p>

    <!-- Search -->
    <div class="card">
      <div class="row">
        <div class="grow">
          <label>公司名称 或 LinkedIn 链接</label>
          <input type="text" v-model="rawInput" placeholder="例：Stripe、https://www.linkedin.com/in/username/"
            @keyup.enter="analyze" @input="onInputChange" />
          <div v-if="inputHint" style="margin-top:4px;font-size:12px" :style="`color:${inputHint.color}`">
            {{ inputHint.text }}
          </div>
        </div>
        <div style="padding-top:22px">
          <button class="btn btn-primary" @click="analyze" :disabled="loading || !validInput">
            {{ loading ? '分析中…' : '开始分析' }}
          </button>
        </div>
      </div>

      <div style="margin-top:10px">
        <span v-for="s in sampleCompanies" :key="s" class="suggest-badge"
          @click="rawInput=s;onInputChange();analyze()">{{ s }}</span>
      </div>
      <div v-if="apiStatus.length" style="margin-top:12px;display:flex;flex-wrap:wrap;gap:6px">
        <span v-for="s in apiStatus" :key="s.name" class="api-badge"
          :class="s.ok?'api-ok':'api-err'">
          {{ s.ok ? '✓' : '✗' }} {{ s.name }}
        </span>
      </div>
    </div>

    <template v-if="loaded">
      <!-- Personal Profile Card (NEW) -->
      <div class="card" v-if="personalProfile">
        <div class="card-title">个人档案</div>
        <div style="display:flex;gap:16px;align-items:flex-start;flex-wrap:wrap">
          <div v-if="personalProfile.profilePicture" style="flex-shrink:0">
            <img :src="personalProfile.profilePicture" :alt="personalProfile.fullName"
              style="width:80px;height:80px;border-radius:12px;object-fit:cover;border:1px solid var(--gray-200)"
              @error="$event.target.style.display='none'" />
          </div>
          <div style="flex:1;min-width:200px">
            <div style="font-size:18px;font-weight:600;color:var(--gray-900)">{{ personalProfile.fullName }}</div>
            <div style="margin-top:4px;font-size:13px;color:var(--gray-600)">{{ personalProfile.headline }}</div>
            <div style="margin-top:4px;font-size:12px;color:var(--gray-500)">{{ personalProfile.location }}</div>
            <div style="margin-top:8px;display:flex;flex-wrap:wrap;gap:8px">
              <a v-if="personalProfile.publicProfileUrl" :href="personalProfile.publicProfileUrl"
                target="_blank" rel="noopener" class="ext-link ext-link-blue">
                🔗 LinkedIn 主页
              </a>
              <span v-if="personalProfile.followers" class="ext-link">👥 {{ personalProfile.followers }} 关注者</span>
            </div>
            <div v-if="personalProfile.summary" style="margin-top:10px;font-size:13px;color:var(--gray-600);line-height:1.6;white-space:pre-wrap">{{ personalProfile.summary }}</div>
          </div>
        </div>

        <!-- Experience -->
        <div v-if="personalProfile.experiences && personalProfile.experiences.length" style="margin-top:16px">
          <div class="section-label" style="color:#1e40af">工作经历</div>
          <div v-for="exp in personalProfile.experiences.slice(0,5)" :key="exp.company || exp.title"
            style="padding:6px 0;border-bottom:0.5px solid var(--gray-100);font-size:13px">
            <div style="font-weight:500;color:var(--gray-800)">{{ exp.title }}</div>
            <div style="color:var(--gray-600)">{{ exp.company }} · {{ exp.duration }}</div>
            <div v-if="exp.description" style="margin-top:2px;font-size:12px;color:var(--gray-500)">{{ exp.description.slice(0,120) }}</div>
          </div>
        </div>

        <!-- Skills -->
        <div v-if="personalProfile.skills && personalProfile.skills.length" style="margin-top:12px">
          <div class="section-label" style="color:#92400e">技能标签</div>
          <div class="tag-list">
            <span v-for="s in personalProfile.skills.slice(0,20)" :key="s.name || s"
              class="kw-chip kw-med" style="cursor:pointer" @click="copyWord(s.name || s)">
              {{ s.name || s }} <span v-if="s.endorsements" style="color:var(--gray-400);font-size:10px">×{{ s.endorsements }}</span>
            </span>
          </div>
        </div>

        <!-- Auto-detected company -->
        <div v-if="detectedCompany" style="margin-top:14px;padding:10px;background:#eff6ff;border-radius:8px;border:1px solid #bfdbfe">
          <div style="font-size:12px;color:#1e40af;margin-bottom:6px">📌 从档案中识别到当前/最近公司：</div>
          <div style="display:flex;align-items:center;gap:10px">
            <div style="font-size:14px;font-weight:500;color:#1e40af">{{ detectedCompany }}</div>
            <button class="btn btn-primary" style="padding:4px 12px;font-size:12px"
              @click="analyzeCompany(detectedCompany)">分析该公司</button>
          </div>
        </div>

        <!-- Manual company input (free mode) -->
        <div v-if="showManualInput" style="margin-top:16px;padding:16px;background:linear-gradient(135deg,#fefce8 0%,#fef9c3 100%);border-radius:12px;border:2px solid #f59e0b;box-shadow:0 2px 8px rgba(245,158,11,0.15)">
          <div style="font-size:14px;font-weight:600;color:#92400e;margin-bottom:10px">🏢 请输入公司名称以查看竞品分析</div>
          <p style="font-size:12px;color:#b45309;margin:0 0 12px 0">输入此人所在公司，即可查看该公司的远程职位、技能需求、竞品数据</p>
          <div style="display:flex;gap:10px;flex-wrap:wrap;align-items:center">
            <input v-model="manualCompany" placeholder="请输入公司名称，如 Stripe、Tencent、Alibaba"
              style="flex:1;min-width:200px;padding:10px 14px;border:2px solid #fbbf24;border-radius:8px;font-size:14px;outline:none;transition:border-color 0.2s"
              @keyup.enter="submitManualCompany()" />
            <button class="btn btn-primary" style="padding:10px 20px;font-size:13px;font-weight:500"
              :disabled="!manualCompany.trim()" @click="submitManualCompany()">🔍 开始分析</button>
          </div>
          <div v-if="suggestedCompanies.length" style="margin-top:12px">
            <div style="font-size:11px;color:#92400e;margin-bottom:6px">💡 或者点击匹配的公司（基于姓名推测）：</div>
            <div style="display:flex;flex-wrap:wrap;gap:8px">
              <button v-for="c in suggestedCompanies" :key="c.name || c.domain"
                class="btn"
                style="padding:6px 12px;font-size:12px;background:#fff;border:1px solid #f59e0b;color:#92400e;border-radius:6px;cursor:pointer"
                @click="manualCompany=c.name; submitManualCompany()">
                {{ c.name }}
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- Company Profile Card -->
      <div class="card" v-if="companyProfile">
        <div class="card-title">公司概览 {{ personalProfile ? '（关联公司）' : '' }}</div>
        <div style="display:flex;gap:20px;align-items:flex-start;flex-wrap:wrap">
          <div v-if="companyProfile.logo" style="flex-shrink:0">
            <img :src="companyProfile.logo" :alt="companyProfile.name"
              style="width:80px;height:80px;border-radius:12px;object-fit:cover;border:1px solid var(--gray-200)"
              @error="$event.target.style.display='none'" />
          </div>
          <div style="flex:1;min-width:200px">
            <div style="font-size:18px;font-weight:600;color:var(--gray-900)">{{ companyProfile.name }}</div>
            <div style="margin-top:6px;display:flex;flex-wrap:wrap;gap:8px">
              <a v-if="companyProfile.domain" :href="'https://'+companyProfile.domain"
                target="_blank" rel="noopener" class="ext-link">
                🌐 {{ companyProfile.domain }}
              </a>
              <a v-if="companyProfile.linkedin"
                :href="companyProfile.linkedin" target="_blank" rel="noopener" class="ext-link ext-link-blue">
                💼 LinkedIn 主页
              </a>
            </div>
            <div v-if="companyProfile.description" style="margin-top:8px;font-size:13px;color:var(--gray-600);line-height:1.6">
              {{ companyProfile.description }}
            </div>
          </div>
        </div>
      </div>

      <!-- Key Metrics -->
      <div class="card">
        <div class="card-title">数据总览</div>
        <div style="display:grid;grid-template-columns:repeat(auto-fit,minmax(100px,1fr));gap:10px">
          <div class="metric-card">
            <div class="metric-label">职位总数</div>
            <div class="metric-value">{{ matchedJobs.length }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">职位标题数</div>
            <div class="metric-value">{{ uniqueTitles }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">技能标签</div>
            <div class="metric-value">{{ topSkills.length }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">有薪资信息</div>
            <div class="metric-value">{{ jobsWithSalary }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">数据来源</div>
            <div class="metric-value" style="font-size:12px">{{ dataSource }}</div>
          </div>
        </div>
      </div>

      <!-- Top Skills -->
      <div class="card">
        <div class="card-title">核心技能关键词 TOP 20</div>
        <p style="font-size:12px;color:var(--gray-600);margin-bottom:12px">从职位标签中提取，反映该公司的人才需求重心</p>
        <div v-for="tag in topSkills.slice(0, 20)" :key="tag.name"
          style="display:flex;align-items:center;gap:8px;padding:3px 0">
          <div style="width:100px;font-size:12px;text-align:right;flex-shrink:0;white-space:nowrap;overflow:hidden;text-overflow:ellipsis" :title="tag.name">{{ tag.name }}</div>
          <div style="flex:1;height:6px;background:var(--gray-100);border-radius:3px;overflow:hidden">
            <div :style="`width:${tag.pct}%;height:100%;border-radius:3px;background:hsl(${160+tag.rank*12},60%,${38+tag.ratio*25}%)`"></div>
          </div>
          <div style="width:28px;font-size:11px;color:var(--gray-600);text-align:right">{{ tag.count }}</div>
        </div>
        <div v-if="!topSkills.length" style="color:var(--gray-400);font-size:13px;padding:12px 0">
          暂无技能标签数据
        </div>
      </div>

      <!-- Skill Categorization -->
      <div class="card" v-if="techSkills.length || softSkills.length">
        <div class="card-title">技能分类洞察</div>
        <div class="grid-2">
          <div>
            <div class="section-label" style="color:#1e40af">技术 / 工具类 ({{ techSkills.length }})</div>
            <div class="tag-list" v-if="techSkills.length">
              <span v-for="s in techSkills" :key="s" class="kw-chip kw-med" style="cursor:pointer" @click="copyWord(s)">{{ s }}</span>
            </div>
            <div v-else style="font-size:12px;color:var(--gray-400)">未检测到</div>
          </div>
          <div>
            <div class="section-label" style="color:#92400e">商业 / 软技能类 ({{ softSkills.length }})</div>
            <div class="tag-list" v-if="softSkills.length">
              <span v-for="s in softSkills" :key="s" class="kw-chip kw-high" style="cursor:pointer" @click="copyWord(s)">{{ s }}</span>
            </div>
            <div v-else style="font-size:12px;color:var(--gray-400)">未检测到</div>
          </div>
        </div>
      </div>

      <!-- Role Distribution -->
      <div class="card" v-if="roleStats.length">
        <div class="card-title">职位类型分布 TOP 10</div>
        <div v-for="role in roleStats" :key="role.title"
          style="display:flex;align-items:center;gap:8px;padding:4px 0">
          <div style="width:180px;font-size:12px;flex-shrink:0;white-space:nowrap;overflow:hidden;text-overflow:ellipsis" :title="role.title">{{ role.title }}</div>
          <div style="flex:1;height:6px;background:var(--gray-100);border-radius:3px;overflow:hidden">
            <div :style="`width:${role.pct}%;height:100%;border-radius:3px;background:var(--blue)`"></div>
          </div>
          <div style="width:28px;font-size:11px;color:var(--gray-600);text-align:right">{{ role.count }}</div>
        </div>
      </div>

      <!-- Job Type -->
      <div class="card" v-if="jobTypeStats.length">
        <div class="card-title">雇佣类型分布</div>
        <div class="row" style="flex-wrap:wrap;gap:16px">
          <div v-for="jt in jobTypeStats" :key="jt.type" style="display:flex;align-items:center;gap:8px">
            <span class="type-badge">{{ jt.type }}</span>
            <div style="width:80px;height:6px;background:var(--gray-100);border-radius:3px;overflow:hidden">
              <div :style="`width:${jt.pct}%;height:100%;border-radius:3px;background:var(--blue)`"></div>
            </div>
            <span style="font-size:12px;color:var(--gray-600)">{{ jt.count }} ({{ jt.pct }}%)</span>
          </div>
        </div>
      </div>

      <!-- Category Distribution -->
      <div class="card" v-if="categoryStats.length">
        <div class="card-title">职能类别分布</div>
        <div style="display:flex;flex-wrap:wrap;gap:8px">
          <div v-for="cat in categoryStats" :key="cat.name" class="cat-chip">
            <span style="font-weight:500">{{ cat.name }}</span>
            <span style="color:var(--gray-500)">{{ cat.count }}</span>
          </div>
        </div>
      </div>

      <!-- Salary Insights -->
      <div class="card" v-if="salaryInsights.length">
        <div class="card-title">薪资洞察</div>
        <div v-for="si in salaryInsights" :key="si.title" class="salary-item">
          <div style="font-size:13px;font-weight:500;color:var(--gray-800)">{{ si.title }}</div>
          <div style="font-size:12px;color:var(--gray-600)">{{ si.salary }}</div>
        </div>
      </div>

      <!-- Company Suggestions from Clearbit -->
      <div class="card" v-if="suggestedCompanies.length">
        <div class="card-title">相关公司推荐</div>
        <p style="font-size:12px;color:var(--gray-600);margin-bottom:12px">基于名称相似度推荐，点击可切换分析</p>
        <div style="display:flex;flex-wrap:wrap;gap:8px">
          <div v-for="c in suggestedCompanies" :key="c.domain" class="company-chip"
            @click="rawInput=c.name;onInputChange();analyze()" :title="'域名: '+c.domain">
            <img v-if="c.logo" :src="c.logo" style="width:20px;height:20px;border-radius:4px" @error="$event.target.style.display='none'" />
            <span>{{ c.name }}</span>
          </div>
        </div>
      </div>

      <!-- Actionable insights -->
      <div class="card">
        <div class="card-title">策略建议</div>
        <div v-for="(insight, i) in insights" :key="i"
          style="display:flex;gap:10px;padding:8px 0;border-bottom:0.5px solid var(--gray-100)">
          <span style="font-size:14px;flex-shrink:0">{{ insight.icon }}</span>
          <div style="font-size:13px;color:var(--gray-800);line-height:1.6">{{ insight.text }}</div>
        </div>
      </div>

      <!-- Jobs list -->
      <div class="card">
        <div class="card-title">相关职位列表 ({{ matchedJobs.length }})</div>
        <div v-if="matchedJobs.length">
          <div style="margin-bottom:10px">
            <input type="text" v-model="jobSearch" placeholder="搜索职位标题..."
              style="width:100%;max-width:300px;padding:6px 10px;border:1px solid var(--gray-200);border-radius:6px;font-size:12px" />
          </div>
          <div v-for="job in filteredJobs" :key="job.id" class="job-item">
            <div style="display:flex;align-items:center;gap:10px;margin-bottom:4px">
              <img v-if="job.company_logo || job.company_logo_url" :src="job.company_logo || job.company_logo_url"
                style="width:32px;height:32px;border-radius:6px;object-fit:cover"
                loading="lazy" @error="$event.target.style.display='none'" />
              <div style="flex:1;min-width:0">
                <a :href="job.url" target="_blank" rel="noopener"
                  style="font-size:13px;font-weight:500;color:var(--blue);text-decoration:none;display:block;white-space:nowrap;overflow:hidden;text-overflow:ellipsis">{{ job.title }}</a>
                <div style="font-size:11px;color:var(--gray-600)">
                  {{ job.company_name || '' }} · {{ job.salary || '薪资未公开' }} · {{ formatJobType(job.job_type) }} · {{ formatDate(job.publication_date) }}
                </div>
                <div v-if="job.candidate_required_location" style="font-size:11px;color:var(--gray-500)">
                  📍 {{ job.candidate_required_location }}
                </div>
              </div>
              <div v-if="job.category" class="cat-badge">{{ job.category }}</div>
            </div>
            <div class="tag-list" style="margin:0">
              <span v-for="tag in (job.tags||[]).slice(0,8)" :key="tag" class="kw-chip kw-med" style="font-size:11px;padding:1px 7px">{{ tag }}</span>
            </div>
          </div>
        </div>
        <p v-else style="color:var(--gray-400);font-size:13px">
          没有找到匹配的职位数据。可能原因：该公司不招远程职位，或公司名称需要调整。建议尝试使用上方推荐的公司名称。
        </p>
      </div>

      <!-- Export -->
      <div class="card" style="text-align:center">
        <button class="btn" @click="exportReport" :disabled="!matchedJobs.length">📄 导出分析报告</button>
      </div>
    </template>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

// ─── State ───
const rawInput = ref('')
const loading = ref(false)
const loaded = ref(false)
const matchedJobs = ref([])
const companyProfile = ref(null)
const suggestedCompanies = ref([])
const apiStatus = ref([])
const jobSearch = ref('')
const personalProfile = ref(null)
const detectedCompany = ref('')
const inputHint = ref(null)
const showManualInput = ref(false)
const manualCompany = ref('')
const WORKER_ENDPOINT = '' // <- 部署 Cloudflare Worker 后，填入 Worker URL（例如 https://xxx.workers.dev）
// 免费替代方案：Clearbit Autocomplete API（无需 Key）+ 本地模拟数据降级
const USE_FREE_MODE = true // 设为 false 则走 Worker（需要部署 Cloudflare Worker）
const validInput = computed(() => {
  const v = rawInput.value.trim()
  return v.length > 0 && inputHint.value?.type !== 'invalid'
})

const sampleCompanies = ['Stripe', 'Notion', 'Vercel', 'Shopify', 'GitLab', 'Deel', 'Webflow', 'OpenAI']


// ─── Input change detection ───
function onInputChange() {
  const v = rawInput.value.trim()
  if (!v) {
    inputHint.value = null
    return
  }
  // Detect LinkedIn URL
  if (/linkedin\.com/i.test(v)) {
    if (/\/in\//i.test(v)) {
      inputHint.value = { type: 'personal', color: '#1e40af', text: '👤 已识别 LinkedIn 个人主页链接，将采集个人档案数据' }
    } else if (/\/company\//i.test(v)) {
      inputHint.value = { type: 'company-url', color: '#059669', text: '🏢 已识别 LinkedIn 公司主页链接，将提取公司名称' }
    } else {
      inputHint.value = { type: 'linkedin-other', color: '#d97706', text: '⚠️ 无法识别 LinkedIn URL 格式，将作为公司名称搜索' }
    }
  } else {
    inputHint.value = { type: 'company', color: '#059669', text: '🏢 将以公司名称进行搜索' }
  }
}

// ─── URL Parsing ───
function parseLinkedInURL(url) {
  try {
    // Use URL constructor directly (it handles encoded URLs correctly)
    const parsed = new URL(url)
    if (!parsed.hostname.includes('linkedin.com')) return null

    const parts = parsed.pathname.split('/').filter(Boolean)
    // linkedin.com/in/username
    if (parts[0] === 'in' && parts[1]) {
      const slug = parts[1]  // keep encoded slug for API use
      // Build display name by decoding the slug first
      const decodedSlug = decodeURIComponent(slug)
      const nameParts = decodedSlug.split('-').filter(p => p.length > 0)
      // Remove trailing LinkedIn ID (all digits, or alphanumeric ID like "a3158112a")
      const displayName = nameParts.filter((p, i) => {
        if (i === nameParts.length - 1 && /^[a-z]+\d+[a-z]+$/.test(p)) return false
        if (/^\d+$/.test(p)) return false
        return true
      }).join(' ')
      return { type: 'personal', slug, displayName }
    }
    // linkedin.com/company/company-name
    if (parts[0] === 'company' && parts[1]) {
      let name = decodeURIComponent(parts[1]).replace(/-/g, ' ')
      // Clean common Chinese company suffixes for better API matching
      name = name.replace(/\s*(有限公司|股份有限公司|有限责任公司|集团|控股|科技|技术|网络|信息|咨询|服务|管理|投资|发展|实业|国际|中国|美国|全球)\s*/g, ' ').trim()
      name = name.replace(/\s+/g, ' ')
      return { type: 'company', name }
    }
  } catch {}
  return null
}

// ─── Tech keyword database (expanded) ───
const TECH_KEYWORDS = new Set([
  'python','javascript','typescript','react','vue','node','node.js','docker','kubernetes','aws','gcp','azure',
  'sql','postgresql','redis','mongodb','graphql','rest api','ci/cd','terraform','linux',
  'figma','sketch','tailwind','html','css','java','go','golang','rust','swift','kotlin','flutter',
  'tensorflow','pytorch','spark','airflow','kafka','elasticsearch','tableau','power bi',
  'c++','c#','ruby','rails','django','flask','spring','angular','next.js','nuxt',
  'machine learning','data engineering','data science','devops','sre',
  'saas','api','microservices','serverless','blockchain','ai/ml','deep learning',
  '.net','unity','unreal','laravel','symfony','svelte','solidity',
  'react native','data analysis','automation',
  'gcp','data engineering','shopify','video','salesforce','hubspot',
  'snowflake','databricks','bigquery','redshift','neo4j','cassandra',
  'jenkins','github actions','argocd','ansible','puppet','chef',
  'webpack','vite','rollup','babel','eslint','jest','cypress','playwright',
  'react.js','vue.js','express','nestjs','fastapi','gin',
])

// ─── API Endpoints ───
const CLEARBIT_SUGGEST = 'https://autocomplete.clearbit.com/v1/companies/suggest'
const REMOTIVE_API = 'https://remotive.com/api/remote-jobs'
const REMOTEOK_API = 'https://remoteok.com/api'

// ─── Chinese company name → English name mapping ───
const CN_COMPANY_MAP = {
  '微软': 'Microsoft', '微软中国': 'Microsoft', '微软中国有限公司': 'Microsoft',
  '腾讯': 'Tencent', '腾讯科技': 'Tencent', '深圳市腾讯计算机系统有限公司': 'Tencent',
  '阿里巴巴': 'Alibaba', '阿里巴巴集团': 'Alibaba', '阿里': 'Alibaba',
  '百度': 'Baidu', '百度在线': 'Baidu', '北京百度网讯科技有限公司': 'Baidu',
  '字节跳动': 'ByteDance', '字节': 'ByteDance', '今日头条': 'ByteDance', '抖音': 'ByteDance', 'tiktok': 'ByteDance',
  '华为': 'Huawei', '华为技术': 'Huawei', '华为技术有限公司': 'Huawei',
  '京东': 'JD.com', '京东集团': 'JD.com', '北京京东世纪贸易有限公司': 'JD.com',
  '美团': 'Meituan', '美团点评': 'Meituan', '北京三快科技有限公司': 'Meituan',
  '拼多多': 'Pinduoduo', '拼多多集团': 'Pinduoduo', '上海寻梦信息技术有限公司': 'Pinduoduo',
  '网易': 'NetEase', '网易公司': 'NetEase', '杭州网易': 'NetEase',
  '小米': 'Xiaomi', '小米科技': 'Xiaomi', '北京小米科技有限责任公司': 'Xiaomi',
  '滴滴': 'DiDi', '滴滴出行': 'DiDi', '北京小桔科技有限公司': 'DiDi',
  '蚂蚁集团': 'Ant Group', '蚂蚁金服': 'Ant Group', '蚂蚁科技集团': 'Ant Group',
  '大疆': 'DJI', '大疆创新': 'DJI', '深圳市大疆创新科技有限公司': 'DJI',
  '比亚迪': 'BYD', '比亚迪股份': 'BYD', '比亚迪股份有限公司': 'BYD',
  '联想': 'Lenovo', '联想集团': 'Lenovo',
  '海尔': 'Haier', '海尔集团': 'Haier',
  '海尔智家': 'Haier Smart Home',
  '中兴': 'ZTE', '中兴通讯': 'ZTE', '中兴通讯股份有限公司': 'ZTE',
  '快手': 'Kuaishou', '快手科技': 'Kuaishou', '北京快手科技有限公司': 'Kuaishou',
  '哔哩哔哩': 'Bilibili', 'b站': 'Bilibili', '上海宽娱': 'Bilibili',
  '携程': 'Trip.com', '携程旅行': 'Trip.com', '携程集团': 'Trip.com',
  '智联招聘': 'Zhaopin', '前程无忧': '51job',
  'oppo': 'OPPO', '欧珀': 'OPPO', '广东欧珀': 'OPPO',
  'vivo': 'vivo', '维沃': 'vivo', '广东步步高': 'vivo',
  '商汤': 'SenseTime', '商汤科技': 'SenseTime',
  '科大讯飞': 'iFlytek', '科大讯飞股份': 'iFlytek',
  '海康威视': 'Hikvision', '杭州海康威视': 'Hikvision',
  '大华': 'Dahua', '浙江大华': 'Dahua',
  '中芯国际': 'SMIC', '中芯国际集成电路': 'SMIC',
  '格力': 'Gree', '格力电器': 'Gree', '珠海格力': 'Gree',
  '中国平安': 'Ping An', '平安集团': 'Ping An',
  '工商银行': 'ICBC', '中国工商银行': 'ICBC',
  '建设银行': 'CCB', '中国建设银行': 'CCB',
  '招商银行': 'CMB', '招商银行股份': 'CMB',
  '中国银行': 'Bank of China', '中国银行股份': 'Bank of China',
  '中国移动': 'China Mobile', '中国电信': 'China Telecom', '中国联通': 'China Unicom',
  '国家电网': 'State Grid',
  '中国石油': 'CNPC', '中国石化': 'Sinopec',
  '蚂蚁': 'Ant Group', '微众银行': 'WeBank',
  'Shopee': 'Shopee', '虾皮': 'Shopee',
  'Lazada': 'Lazada', '来赞达': 'Lazada',
  'Grab': 'Grab', 'GrabTaxi': 'Grab',
  'Google': 'Google', '谷歌': 'Google', '谷歌中国': 'Google',
  'Apple': 'Apple', '苹果': 'Apple', '苹果公司': 'Apple', '苹果中国': 'Apple',
  'Amazon': 'Amazon', '亚马逊': 'Amazon', '亚马逊中国': 'Amazon',
  'Meta': 'Meta', 'Facebook': 'Meta', '脸书': 'Meta', '元': 'Meta',
  'Netflix': 'Netflix', '奈飞': 'Netflix',
  'Tesla': 'Tesla', '特斯拉': 'Tesla', '特斯拉中国': 'Tesla',
  'IBM': 'IBM', '国际商业机器': 'IBM',
  'Intel': 'Intel', '英特尔': 'Intel',
  'Samsung': 'Samsung', '三星': 'Samsung', '三星电子': 'Samsung',
  'Sony': 'Sony', '索尼': 'Sony',
  'Oracle': 'Oracle', '甲骨文': 'Oracle',
  'SAP': 'SAP', '思爱普': 'SAP',
  'Adobe': 'Adobe',
  'Salesforce': 'Salesforce',
  'Shopify': 'Shopify',
  'Stripe': 'Stripe',
  'Spotify': 'Spotify', '声田': 'Spotify',
  'Airbnb': 'Airbnb', '爱彼迎': 'Airbnb',
  'Uber': 'Uber', '优步': 'Uber',
  'LinkedIn': 'LinkedIn', '领英': 'LinkedIn',
  'Twitter': 'X Corp', '推特': 'X Corp', 'X': 'X Corp',
  'Zoom': 'Zoom',
  'Slack': 'Slack',
  'Notion': 'Notion',
  'Vercel': 'Vercel',
  'OpenAI': 'OpenAI',
  'NVIDIA': 'NVIDIA', '英伟达': 'NVIDIA', '英伟达中国': 'NVIDIA',
  '阿里云': 'Alibaba Cloud', '华为云': 'Huawei Cloud', '腾讯云': 'Tencent Cloud',
  '百度云': 'Baidu Cloud', '金山云': 'Kingsoft Cloud',
  '美团': 'Meituan', '饿了么': 'Ele.me', '飞猪': 'Fliggy',
  '微信': 'WeChat', '企业微信': 'WeCom', '钉钉': 'DingTalk',
  '高德': 'Amap', '高德地图': 'Amap',
  '得物': 'Dewu', '得物App': 'Dewu',
  'Shein': 'SHEIN', '希音': 'SHEIN',
  'Temu': 'Temu',
  'TikTok': 'TikTok',
  '中国': '',  // generic fallback, empty = skip
  '有限公司': '',  // suffix, empty = skip
  '集团': '',  // suffix
  '科技': '',  // suffix
  '有限': '',  // suffix
  '公司': '',  // suffix
}

// Resolve a Chinese company name to its English equivalent for API queries
function resolveCompanyQuery(name) {
  const trimmed = name.trim()
  if (!trimmed) return { english: trimmed, original: trimmed }
  
  // Direct lookup
  if (CN_COMPANY_MAP[trimmed]) {
    const en = CN_COMPANY_MAP[trimmed]
    if (en) return { english: en, original: trimmed }
  }
  
  // Try each segment of the name (handles "微软 中国 有限公司" → "微软" → "Microsoft")
  const segments = trimmed.split(/[\s\-·]+/).filter(Boolean)
  for (const seg of segments) {
    if (CN_COMPANY_MAP[seg] && CN_COMPANY_MAP[seg]) {
      return { english: CN_COMPANY_MAP[seg], original: trimmed }
    }
  }
  
  // Check if name already contains English (mixed names)
  if (/[a-zA-Z]/.test(trimmed)) {
    // Extract English parts
    const englishPart = trimmed.replace(/[一-鿿　-〿＀-￯]+/g, '').trim()
    if (englishPart.length >= 2) {
      return { english: englishPart, original: trimmed }
    }
  }
  
  return { english: trimmed, original: trimmed }
}

// ─── Main Analysis ───
async function analyze() {
  const v = rawInput.value.trim()
  if (!v || loading.value) return
  loading.value = true
  loaded.value = false
  matchedJobs.value = []
  companyProfile.value = null
  suggestedCompanies.value = []
  personalProfile.value = null
  detectedCompany.value = ''
  apiStatus.value = []

  try {
    const linkedinURL = parseLinkedInURL(v)

    if (linkedinURL && linkedinURL.type === 'personal') {
      // Personal LinkedIn URL → fetch profile, then analyze their company
      await analyzePersonalURL(linkedinURL)
    } else {
      // Company name or company LinkedIn URL
      const companyName = linkedinURL
        ? linkedinURL.name
        : v

      await analyzeCompany(companyName)
    }

    loaded.value = true
  } catch (err) {
    console.error('Analysis error:', err)
    alert('分析过程中出错: ' + err.message)
  } finally {
    loading.value = false
  }
}

// ─── Analyze personal URL (free mode) ───
async function analyzePersonalURL(parsed) {
  try {
    const profile = await fetchPersonalProfile(parsed.slug)
    personalProfile.value = profile
    apiStatus.value.push({ name: 'LinkedIn 个人档案', ok: true })

    // Free mode: need manual company input
    if (profile._needsManualInput) {
      showManualInput.value = true
      try {
        const res = await fetch(`${CLEARBIT_SUGGEST}?query=${encodeURIComponent(profile.fullName)}`)
        if (res.ok) {
          const data = await res.json()
          if (data && data.length > 0) {
            suggestedCompanies.value = data.slice(0, 6)
          }
        }
      } catch {}
      return
    }

    // Try to detect current company from experience
    if (profile.experiences && profile.experiences.length > 0) {
      const current = profile.experiences.find(e => !e.endDate || e.current)
        || profile.experiences[0]
      if (current && current.company) {
        detectedCompany.value = current.company
        await analyzeCompany(current.company)
        return
      }
    }
  } catch (err) {
    console.warn('Profile fetch failed:', err)
    apiStatus.value.push({ name: 'LinkedIn 个人档案', ok: false })
  }

  // Fallback: show company suggestions
  if (!detectedCompany.value) {
    try {
      const q = parsed.displayName
      const res = await fetch(`${CLEARBIT_SUGGEST}?query=${encodeURIComponent(q)}`)
      if (res.ok) {
        const data = await res.json()
        if (data && data.length > 0) {
          suggestedCompanies.value = data.slice(0, 6)
        }
      }
    } catch {}
  }
}

// ─── Submit manual company (free mode) ───
async function submitManualCompany() {
  const name = manualCompany.value.trim()
  if (!name) return
  showManualInput.value = false
  detectedCompany.value = name
  await analyzeCompany(name)
}

// ─── Analyze company (core logic) ───
async function analyzeCompany(name) {
  const q = name.trim()
  if (!q) return

  // Resolve Chinese company name to English for API queries
  const { english: enName, original: cnName } = resolveCompanyQuery(q)
  const queryName = enName || q
  const usedTranslation = enName && enName.toLowerCase() !== q.toLowerCase()

  const [clearbitResult, remotiveResult, remoteokResult] = await Promise.allSettled([
    fetchCompanyProfile(queryName, q),
    fetchJobsFromRemotive(),
    fetchJobsFromRemoteOK(),
  ])

  // Process Clearbit
  if (clearbitResult.status === 'fulfilled' && clearbitResult.value) {
    companyProfile.value = clearbitResult.value.profile
    if (!personalProfile.value) {
      suggestedCompanies.value = clearbitResult.value.suggestions
    }
    apiStatus.value.push({ name: usedTranslation ? `Clearbit 公司信息（已翻译: ${q} → ${enName}）` : 'Clearbit 公司信息', ok: true })
  } else {
    apiStatus.value.push({ name: 'Clearbit 公司信息', ok: false })
  }

  // Merge & filter jobs from all sources
  let allJobs = []
  if (remotiveResult.status === 'fulfilled') allJobs.push(...remotiveResult.value)
  if (remoteokResult.status === 'fulfilled') allJobs.push(...remoteokResult.value)

  // Deduplicate by id
  const seen = new Set()
  allJobs = allJobs.filter(j => {
    if (seen.has(j.id)) return false
    seen.add(j.id)
    return true
  })

  // Smart matching: score each job by relevance
  const scored = allJobs.map(job => {
    let score = 0
    const company = (job.company_name || '').toLowerCase()
    const title = (job.title || '').toLowerCase()
    const tags = (job.tags || []).join(' ').toLowerCase()
    const all = company + ' ' + title + ' ' + tags

    // Exact company name match (highest priority)
    if (queryName && company === queryName.toLowerCase()) score += 100
    if (q && company === q.toLowerCase()) score += 100

    // Company contains query
    if (queryName && company.includes(queryName.toLowerCase())) score += 50
    if (q && company.includes(q.toLowerCase())) score += 50

    // Any word from query matches company
    if (queryName) {
      const words = queryName.toLowerCase().split(/[\s\-]+/).filter(w => w.length > 2)
      for (const w of words) {
        if (company.includes(w)) score += 30
        if (title.includes(w)) score += 15
        if (tags.includes(w)) score += 10
      }
    }

    // Original name words (for Chinese)
    if (q && q !== queryName) {
      const origWords = q.toLowerCase().split(/[\s\-·]+/).filter(w => w.length > 1)
      for (const w of origWords) {
        if (company.includes(w)) score += 20
        if (title.includes(w)) score += 10
      }
    }

    job._relevanceScore = score
    return job
  })

  // Sort by relevance, but show all jobs (not just matched ones)
  const matched = scored.filter(j => j._relevanceScore >= 30).sort((a, b) => b._relevanceScore - a._relevanceScore)
  const others = scored.filter(j => j._relevanceScore < 30)

  // Use matched jobs if any; otherwise show all jobs as "unfiltered"
  if (matched.length > 0) {
    matchedJobs.value = matched
  } else {
    // No exact match — show all jobs with relevance hint
    matchedJobs.value = scored.sort((a, b) => b._relevanceScore - a._relevanceScore).slice(0, 50)
  }

  // API status
  if (matched.length > 0) {
    apiStatus.value.push({ name: `职位匹配 (${matched.length} 条精准匹配 / ${allJobs.length} 总职位)`, ok: true })
  } else if (allJobs.length > 0) {
    apiStatus.value.push({ name: `职位数据 (${allJobs.length} 条，未找到精确匹配)`, ok: true })
  } else {
    apiStatus.value.push({ name: '职位数据', ok: false })
  }
}

// ─── Fetch personal profile (Free Mode: manual input / mock data) ───
async function fetchPersonalProfile(username) {
  // Try Cloudflare Worker first (if configured)
  if (WORKER_ENDPOINT) {
    try {
      const url = `${WORKER_ENDPOINT}?username=${encodeURIComponent(username)}`
      const res = await fetch(url)
      if (res.ok) {
        const data = await res.json()
        return normalizeProfileData(data)
      }
    } catch (e) {
      console.warn('Worker fetch failed, falling back to manual mode:', e.message)
    }
  }

  // Free fallback: decode username for display, trigger manual input UI
  let displayName = username
  try { displayName = decodeURIComponent(username) } catch {}
  // Clean up: remove trailing LinkedIn ID (e.g. "a3158112a")
  const nameParts = displayName.split('-').filter(p => p.length > 0)
  const cleanName = nameParts.filter((p, i) => {
    if (i === nameParts.length - 1 && /^[a-z]+\d+[a-z]+$/i.test(p)) return false
    if (/^\d+$/.test(p)) return false
    return true
  }).join(' ')

  return {
    fullName: cleanName || displayName.replace(/-/g, ' '),
    headline: '',
    location: '',
    summary: '💡 当前为免费模式，显示基础档案信息。输入公司名称可查看完整竞品分析。',
    profilePicture: '',
    publicProfileUrl: `https://www.linkedin.com/in/${username}/`,
    followers: 0,
    connections: 0,
    experiences: [],
    education: [],
    skills: [],
    _needsManualInput: true,
  }
}

// ─── Normalize RapidAPI response ───
function normalizeProfileData(data) {
  // The API may return data in different shapes; normalize to a consistent format
  const profile = data.profile || data.data || data
  return {
    fullName: profile.fullName || profile.full_name || profile.name || 'Unknown',
    headline: profile.headline || profile.title || '',
    location: profile.location || profile.geo || '',
    summary: profile.summary || profile.about || profile.bio || '',
    profilePicture: profile.profilePicture || profile.avatar || profile.photo || '',
    publicProfileUrl: profile.publicProfileUrl || profile.url || '',
    followers: profile.followers || profile.followerCount || 0,
    connections: profile.connections || profile.connectionCount || 0,
    experiences: normalizeExperiences(profile.experiences || profile.experience || []),
    education: profile.education || [],
    skills: normalizeSkills(profile.skills || []),
  }
}

function normalizeExperiences(exp) {
  if (!Array.isArray(exp)) return []
  return exp.map(e => ({
    title: e.title || e.role || e.position || '',
    company: e.company || e.companyName || '',
    duration: e.duration || e.date || '',
    description: e.description || e.summary || '',
    current: e.current || e.isCurrent || false,
  }))
}

function normalizeSkills(skills) {
  if (!Array.isArray(skills)) return []
  return skills.map(s => ({
    name: typeof s === 'string' ? s : (s.name || s.skill || ''),
    endorsements: s.endorsements || s.count || 0,
  })).filter(s => s.name)
}

// ─── Clearbit: Company profile + suggestions ───
async function fetchCompanyProfile(enName, originalName) {
  // Try English name first, then original as fallback
  const queries = [enName]
  if (originalName && originalName !== enName) queries.push(originalName)
  
  for (const q of queries) {
    try {
      const res = await fetch(`${CLEARBIT_SUGGEST}?query=${encodeURIComponent(q)}`)
      if (!res.ok) continue
      const data = await res.json()
      if (!data || !data.length) continue

      const best = data[0]
      const profile = {
        name: best.name,
        domain: best.domain,
        logo: best.logo || `https://logo.clearbit.com/${best.domain}`,
        linkedin: best.linkedin
          ? (best.linkedin.startsWith('http') ? best.linkedin : `https://www.linkedin.com${best.linkedin}`)
          : best.domain ? `https://www.linkedin.com/company/${best.domain.split('.')[0]}/` : '',
        description: best.description || `官网: ${best.domain}`,
      }
      const suggestions = data.slice(1, 8).filter(Boolean)
      return { profile, suggestions }
    } catch { continue }
  }
  return null
}

// ─── Remotive: Remote jobs (fetch ALL — server-side search is broken) ───
async function fetchJobsFromRemotive() {
  try {
    const res = await fetch(REMOTIVE_API)
    if (!res.ok) return []
    const data = await res.json()
    return (data.jobs || []).map(j => ({
      id: `remotive-${j.id}`,
      title: j.title || '',
      company_name: j.company_name || '',
      company_logo: j.company_logo || '',
      company_logo_url: j.company_logo_url || '',
      url: j.url || '',
      salary: j.salary || '',
      job_type: j.job_type || '',
      category: j.category || '',
      tags: j.tags || [],
      candidate_required_location: j.candidate_required_location || '',
      publication_date: j.publication_date || '',
    }))
  } catch { return [] }
}

// ─── RemoteOK: Remote jobs (large dataset, ~100 jobs) ───
async function fetchJobsFromRemoteOK() {
  try {
    const res = await fetch(REMOTEOK_API)
    if (!res.ok) return []
    const data = await res.json()
    // First item is metadata, skip it
    if (!Array.isArray(data) || data.length < 2) return []
    return data.slice(1).map(j => ({
      id: `remoteok-${j.id}`,
      title: j.position || '',
      company_name: j.company || '',
      company_logo: j.company_logo || '',
      company_logo_url: j.company_logo || '',
      url: j.url || j.apply_url || `https://remoteok.com/remote-jobs/${j.slug}`,
      salary: j.salary || '',
      job_type: j.type || '',
      category: j.category || '',
      tags: j.tags || [],
      candidate_required_location: j.location || '',
      publication_date: j.date || '',
    }))
  } catch { return [] }
}

// ─── Computed: Skills ───
const topSkills = computed(() => {
  const freq = {}
  matchedJobs.value.forEach(j => {
    (j.tags || []).forEach(t => {
      const k = t.trim()
      if (!k) return
      freq[k] = (freq[k] || 0) + 1
    })
  })
  const maxCount = Math.max(...Object.values(freq), 1)
  return Object.entries(freq)
    .map(([name, count], i) => ({
      name, count,
      ratio: count / maxCount,
      pct: Math.round(count / maxCount * 100),
      rank: i,
    }))
    .sort((a, b) => b.count - a.count)
})

const techSkills = computed(() =>
  topSkills.value.filter(t => TECH_KEYWORDS.has(t.name.toLowerCase())).map(t => t.name).slice(0, 20)
)
const softSkills = computed(() =>
  topSkills.value.filter(t => !TECH_KEYWORDS.has(t.name.toLowerCase())).map(t => t.name).slice(0, 20)
)

const uniqueTitles = computed(() =>
  new Set(matchedJobs.value.map(j => j.title)).size
)

const jobsWithSalary = computed(() =>
  matchedJobs.value.filter(j => j.salary).length
)

const dataSource = computed(() => {
  const sources = []
  if (matchedJobs.value.length) sources.push('Remotive')
  if (companyProfile.value) sources.push('Clearbit')
  if (personalProfile.value) sources.push('LinkedIn 档案')
  return sources.join(' + ') || '无'
})

// ─── Computed: Stats ───
const roleStats = computed(() => {
  const freq = {}
  matchedJobs.value.forEach(j => {
    const t = j.title || 'Unknown'
    freq[t] = (freq[t] || 0) + 1
  })
  const maxCount = Math.max(...Object.values(freq), 1)
  return Object.entries(freq)
    .map(([title, count]) => ({ title, count, pct: Math.round(count / maxCount * 100) }))
    .sort((a, b) => b.count - a.count)
    .slice(0, 10)
})

const jobTypeStats = computed(() => {
  const total = matchedJobs.value.length || 1
  const freq = {}
  matchedJobs.value.forEach(j => {
    const t = j.job_type || 'other'
    freq[t] = (freq[t] || 0) + 1
  })
  return Object.entries(freq)
    .map(([type, count]) => ({ type: formatJobType(type), count, pct: Math.round(count / total * 100) }))
    .sort((a, b) => b.count - a.count)
})

const categoryStats = computed(() => {
  const freq = {}
  matchedJobs.value.forEach(j => {
    const cat = j.category || 'Other'
    freq[cat] = (freq[cat] || 0) + 1
  })
  return Object.entries(freq)
    .map(([name, count]) => ({ name, count }))
    .sort((a, b) => b.count - a.count)
})

const salaryInsights = computed(() => {
  return matchedJobs.value
    .filter(j => j.salary)
    .slice(0, 8)
    .map(j => ({ title: j.title, salary: j.salary }))
})

const filteredJobs = computed(() => {
  if (!jobSearch.value.trim()) return matchedJobs.value
  const q = jobSearch.value.toLowerCase()
  return matchedJobs.value.filter(j =>
    j.title?.toLowerCase().includes(q) ||
    j.company_name?.toLowerCase().includes(q) ||
    (j.tags || []).some(t => t.toLowerCase().includes(q))
  )
})

// ─── Insights engine ───
const insights = computed(() => {
  const items = []
  if (!matchedJobs.value.length && !companyProfile.value && !personalProfile.value) return items

  if (personalProfile.value) {
    items.push({
      icon: '👤',
      text: `正在分析 ${personalProfile.value.fullName} 的档案数据。${detectedCompany.value ? `已自动识别其所在公司「${detectedCompany.value}」并进行分析。` : '请手动输入此人的公司名称以获取竞品分析。'}`
    })
  }

  const top3 = topSkills.value.slice(0, 3).map(t => t.name)
  if (top3.length) {
    items.push({
      icon: '🎯',
      text: `该公司最核心的技能需求是：${top3.join('、')}。在开发信中突出这些关键词能显著提升回复率。`
    })
  }

  const fullTime = jobTypeStats.value.find(j => j.type === '全职' || j.type === 'Full-time')
  if (fullTime) {
    items.push({
      icon: '🏠',
      text: `在 ${matchedJobs.value.length} 个职位中，全职占 ${fullTime.pct}%，说明该公司有稳定的招聘策略。`
    })
  }

  const techCount = techSkills.value.length
  const softCount = softSkills.value.length
  if (techCount > 0) {
    items.push({
      icon: '💻',
      text: `技术技能标签 (${techCount})，` + (softCount > 0 ? `软技能标签 (${softCount})。` : `建议补充软技能关键词以扩大人才池。`)
    })
  }

  if (jobsWithSalary.value === 0 && matchedJobs.value.length > 0) {
    items.push({ icon: '💰', text: '所有职位均未公开薪资信息，薪资范围可能较灵活。' })
  } else if (jobsWithSalary.value > 0) {
    items.push({ icon: '💰', text: `${jobsWithSalary.value} 个职位公开了薪资信息，建议参考这些范围优化沟通策略。` })
  }

  if (categoryStats.value.length > 2) {
    const topCats = categoryStats.value.slice(0, 3).map(c => `${c.name}(${c.count})`).join('、')
    items.push({ icon: '📊', text: `主要招聘方向：${topCats}。` })
  }

  if (companyProfile.value?.domain) {
    items.push({ icon: '🔗', text: `公司官网 ${companyProfile.value.domain}，建议先了解其产品再进行接触。` })
  }

  if (suggestedCompanies.value.length > 0 && !personalProfile.value) {
    items.push({
      icon: '🔄',
      text: `还发现 ${suggestedCompanies.value.length} 个相关公司，可逐一分析扩展竞品地图。`
    })
  }

  if (matchedJobs.value.length === 0 && companyProfile.value && !personalProfile.value) {
    items.push({
      icon: '🔍',
      text: '未找到该公司在远程职位数据库中的记录，但仍是有价值的潜在客户。建议访问其 LinkedIn 主页了解动态。'
    })
  }

  // Check if we have matched vs total
  const hasExactMatch = matchedJobs.value.some(j => (j._relevanceScore || 0) >= 50)
  if (matchedJobs.value.length > 0 && !hasExactMatch) {
    items.push({
      icon: '📊',
      text: '当前显示的职位来自远程工作数据库的最新数据，未找到与该公司精确匹配的职位。你可以浏览这些职位了解市场趋势。'
    })
  }

  return items
})

// ─── Export ───
function exportReport() {
  const lines = [
    `# 竞品 / 同行分析报告`,
    `生成时间: ${new Date().toLocaleString('zh-CN')}`,
    '',
  ]

  if (personalProfile.value) {
    lines.push('## 个人档案', `- 姓名: ${personalProfile.value.fullName}`, `- 职位: ${personalProfile.value.headline}`, `- LinkedIn: ${personalProfile.value.publicProfileUrl || 'N/A'}`, '')
  }
  if (companyProfile.value) {
    lines.push('## 公司概览', `- 名称: ${companyProfile.value.name}`, `- 域名: ${companyProfile.value.domain}`, `- LinkedIn: ${companyProfile.value.linkedin || 'N/A'}`, '')
  }
  lines.push('## 数据总览', `- 匹配职位: ${matchedJobs.value.length}`, `- 职位标题数: ${uniqueTitles.value}`, `- 技能标签数: ${topSkills.value.length}`, '')
  lines.push('## TOP 10 技能关键词', ...topSkills.value.slice(0, 10).map((t, i) => `${i + 1}. ${t.name} (${t.count})`), '')
  lines.push('## 策略建议', ...insights.value.map(ins => `- ${ins.text}`), '')
  lines.push('## 职位列表', ...matchedJobs.value.map(j =>
    `- [${j.title}](${j.url}) | ${j.company_name || ''} | ${j.salary || 'N/A'} | ${j.job_type || 'N/A'}`
  ))

  const blob = new Blob([lines.join('\n')], { type: 'text/markdown;charset=utf-8' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `竞品分析_${rawInput.value.replace(/[^a-zA-Z0-9\u4e00-\u9fa5]/g, '_').slice(0, 30)}_${new Date().toISOString().slice(0, 10)}.md`
  a.click()
  URL.revokeObjectURL(url)
}

// ─── Helpers ───
function formatDate(d) {
  if (!d) return ''
  try { return new Date(d).toLocaleDateString('zh-CN') } catch { return d }
}

function formatJobType(t) {
  if (!t) return '未知'
  const map = {
    full_time: '全职', part_time: '兼职',
    contract: '合同工', freelance: '自由职业', internship: '实习',
  }
  return map[t.toLowerCase()] || t
}

async function copyWord(w) {
  try { await navigator.clipboard.writeText(w) } catch {}
}
</script>

<style scoped>
.ext-link {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  color: var(--gray-600);
  text-decoration: none;
  padding: 3px 8px;
  border: 1px solid var(--gray-200);
  border-radius: 6px;
  transition: all 0.15s;
}
.ext-link:hover {
  background: var(--gray-50);
  border-color: var(--gray-300);
}
.ext-link-blue:hover {
  border-color: #0a66c2;
  color: #0a66c2;
}

.api-badge {
  display: inline-flex;
  align-items: center;
  gap: 3px;
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 4px;
}
.api-ok {
  background: #f0fdf4;
  color: #166534;
  border: 1px solid #bbf7d0;
}
.api-err {
  background: #fef2f2;
  color: #991b1b;
  border: 1px solid #fecaca;
}

.company-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  border: 1px solid var(--gray-200);
  border-radius: 8px;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.15s;
}
.company-chip:hover {
  background: var(--gray-50);
  border-color: var(--blue);
  color: var(--blue);
}

.cat-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 10px;
  background: var(--gray-50);
  border: 1px solid var(--gray-200);
  border-radius: 6px;
  font-size: 12px;
  color: var(--gray-700);
}
.cat-badge {
  font-size: 11px;
  padding: 2px 6px;
  background: #eff6ff;
  color: #1e40af;
  border-radius: 4px;
  flex-shrink: 0;
}

.salary-item {
  padding: 6px 0;
  border-bottom: 0.5px solid var(--gray-100);
}

.job-item {
  padding: 10px 0;
  border-bottom: 0.5px solid var(--gray-100);
}
.job-item:last-child {
  border-bottom: none;
}

.suggest-badge {
  display: inline-block;
  padding: 3px 10px;
  margin: 3px;
  font-size: 12px;
  background: var(--gray-50);
  border: 1px solid var(--gray-200);
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.15s;
}
.suggest-badge:hover {
  background: var(--blue);
  color: white;
  border-color: var(--blue);
}
</style>
