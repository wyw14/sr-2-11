<template>
  <div class="home">
    <div class="stats-row">
      <div class="stat-card">
        <div class="stat-icon">👥</div>
        <div class="stat-content">
          <div class="stat-value">{{ stats.totalUsers }}</div>
          <div class="stat-label">活跃用户</div>
        </div>
      </div>
      <div class="stat-card">
        <div class="stat-icon">📚</div>
        <div class="stat-content">
          <div class="stat-value">{{ stats.totalSkills }}</div>
          <div class="stat-label">技能总数</div>
        </div>
      </div>
      <div class="stat-card">
        <div class="stat-icon">🤝</div>
        <div class="stat-content">
          <div class="stat-value">{{ stats.completedExchanges }}</div>
          <div class="stat-label">完成交换</div>
        </div>
      </div>
      <div class="stat-card">
        <div class="stat-icon">📈</div>
        <div class="stat-content">
          <div class="stat-value">{{ stats.successRate }}%</div>
          <div class="stat-label">成功率</div>
        </div>
      </div>
    </div>

    <div class="main-grid">
      <div class="card">
        <h2 class="section-title">🔥 热门技能</h2>
        <div v-if="popularSkills.length">
          <div v-for="(skill, index) in popularSkills" :key="skill.name" class="skill-rank-item">
            <div class="rank-number" :class="`rank-${index + 1}`">{{ index + 1 }}</div>
            <div class="skill-info">
              <span class="skill-name">{{ skill.name }}</span>
              <div class="skill-counts">
                <span class="count-tag teach">可教 {{ skill.teachCount }}</span>
                <span class="count-tag learn">想学 {{ skill.learnCount }}</span>
              </div>
            </div>
            <div class="skill-demand" :class="skill.demand > 0 ? 'high-demand' : 'low-demand'">
              {{ skill.demand > 0 ? '需求大' : '供应足' }}
            </div>
          </div>
        </div>
        <el-empty v-else description="暂无数据" />
      </div>

      <div class="card">
        <h2 class="section-title">🌟 为你推荐</h2>
        <div v-if="recommendedMatches.length">
          <div v-for="match in recommendedMatches.slice(0, 5)" :key="match.userId" class="match-card">
            <el-avatar :src="match.user.avatar" :size="48" />
            <div class="match-info">
              <div class="match-header">
                <span class="match-name">{{ match.user.username }}</span>
                <span class="score-badge" :class="getScoreClass(match.score)">
                  契合度 {{ match.score }}%
                </span>
              </div>
              <div class="match-skills">
                <span class="skill-tag skill-learn">学: {{ match.matchedSkills.iCanLearn.slice(0, 2).join(', ') }}</span>
                <span class="skill-tag skill-teach">教: {{ match.matchedSkills.iCanTeach.slice(0, 2).join(', ') }}</span>
              </div>
            </div>
            <el-button type="primary" size="small" @click="goToChat(match.userId)">聊一聊</el-button>
          </div>
        </div>
        <el-empty v-else description="暂无匹配，先去发布技能吧">
          <template #action>
            <el-button type="primary" @click="$router.push('/publish')">发布技能</el-button>
          </template>
        </el-empty>
      </div>
    </div>

    <div class="card">
      <div class="discover-header">
        <h2 class="section-title" style="margin: 0">🔍 发现技能搭档</h2>
        <div class="header-actions">
          <el-button v-if="savedFilters.length" type="info" plain size="small" @click="showSavedFilters = !showSavedFilters">
            <el-icon><Collection /></el-icon>已保存筛选 ({{ savedFilters.length }})
          </el-button>
          <el-button type="warning" plain size="small" @click="toggleAdvancedFilter">
            <el-icon v-if="showAdvancedFilter"><ArrowUp /></el-icon>
            <el-icon v-else><ArrowDown /></el-icon>
            {{ showAdvancedFilter ? '收起高级筛选' : '高级筛选' }}
          </el-button>
        </div>
      </div>

      <div v-if="showSavedFilters && savedFilters.length" class="saved-filters-bar">
        <span class="saved-label">快速恢复：</span>
        <div class="saved-tags">
          <el-tag
            v-for="f in savedFilters"
            :key="f.id"
            type="success"
            effect="plain"
            closable
            @click="applySavedFilter(f)"
            @close="handleDeleteFilter(f)"
            class="saved-filter-tag"
          >
            <el-icon style="margin-right: 4px"><Filter /></el-icon>
            {{ f.name }}
          </el-tag>
        </div>
      </div>

      <div class="basic-filter-bar">
        <el-input v-model="filters.keyword" placeholder="搜索用户名、技能或简介" style="width: 220px" clearable @keyup.enter="loadUsers">
          <template #prefix><el-icon><Search /></el-icon></template>
        </el-input>
        <el-select v-model="filters.city" placeholder="选择城市" clearable filterable style="width: 140px">
          <el-option v-for="c in cityList" :key="c" :label="c" :value="c" />
        </el-select>
        <el-select v-model="filters.minRating" placeholder="最低评分" clearable style="width: 120px">
          <el-option label="4.5+ 分" :value="4.5" />
          <el-option label="4+ 分" :value="4" />
          <el-option label="3.5+ 分" :value="3.5" />
          <el-option label="3+ 分" :value="3" />
        </el-select>
        <el-button type="primary" @click="loadUsers">
          <el-icon><Search /></el-icon>搜索
        </el-button>
        <el-button type="success" plain @click="openSaveDialog">
          <el-icon><Star /></el-icon>保存条件
        </el-button>
        <el-button type="default" plain @click="resetFilters">
          <el-icon><Refresh /></el-icon>重置
        </el-button>
      </div>

      <el-collapse-transition>
        <div v-show="showAdvancedFilter" class="advanced-filter-panel">
          <div class="filter-section">
            <div class="filter-section-title">📍 位置信息</div>
            <div class="filter-row">
              <el-select v-model="filters.province" placeholder="省份" clearable style="width: 160px">
                <el-option v-for="p in provinceList" :key="p" :label="p" :value="p" />
              </el-select>
              <el-select v-model="filters.city" placeholder="城市" clearable filterable style="width: 160px">
                <el-option v-for="c in cityList" :key="c" :label="c" :value="c" />
              </el-select>
            </div>
          </div>

          <div class="filter-section">
            <div class="filter-section-title">🎯 评分与经验</div>
            <div class="filter-row">
              <el-select v-model="filters.minRating" placeholder="最低评分" clearable style="width: 160px">
                <el-option label="4.5 分以上" :value="4.5" />
                <el-option label="4 分以上" :value="4" />
                <el-option label="3.5 分以上" :value="3.5" />
                <el-option label="3 分以上" :value="3" />
              </el-select>
              <el-select v-model="filters.minExchangeCount" placeholder="最少交换次数" clearable style="width: 180px">
                <el-option label="1 次以上" :value="1" />
                <el-option label="3 次以上" :value="3" />
                <el-option label="5 次以上" :value="5" />
                <el-option label="10 次以上" :value="10" />
              </el-select>
            </div>
          </div>

          <div class="filter-section">
            <div class="filter-section-title">📚 技能筛选</div>
            <div class="filter-block">
              <div class="filter-block-label">
                <span class="block-icon">🎓</span>对方可教（我想学的）
              </div>
              <div class="filter-row">
                <el-select
                  v-model="filters.teachSkillCategory"
                  placeholder="技能类别"
                  clearable
                  style="width: 180px"
                >
                  <el-option v-for="cat in categories" :key="cat.id" :label="`${cat.icon} ${cat.name}`" :value="cat.id" />
                </el-select>
                <el-input
                  v-model="filters.teachSkill"
                  placeholder="技能名称关键词"
                  style="width: 200px"
                  clearable
                >
                  <template #prefix><el-icon><Search /></el-icon></template>
                </el-input>
              </div>
            </div>
            <div class="filter-block">
              <div class="filter-block-label">
                <span class="block-icon">📖</span>对方想学（我可教的）
              </div>
              <div class="filter-row">
                <el-select
                  v-model="filters.learnSkillCategory"
                  placeholder="技能类别"
                  clearable
                  style="width: 180px"
                >
                  <el-option v-for="cat in categories" :key="cat.id" :label="`${cat.icon} ${cat.name}`" :value="cat.id" />
                </el-select>
                <el-input
                  v-model="filters.learnSkill"
                  placeholder="技能名称关键词"
                  style="width: 200px"
                  clearable
                >
                  <template #prefix><el-icon><Search /></el-icon></template>
                </el-input>
              </div>
            </div>
          </div>

          <div class="filter-section">
            <div class="filter-section-title">🔄 交换方式</div>
            <el-radio-group v-model="filters.exchangeMode">
              <el-radio value="online">线上</el-radio>
              <el-radio value="offline">线下</el-radio>
              <el-radio value="both">不限</el-radio>
            </el-radio-group>
          </div>

          <div class="filter-section">
            <div class="filter-section-title">⏰ 可用时间</div>
            <el-checkbox-group v-model="filters.availableTime">
              <el-checkbox value="weekday_morning">工作日上午</el-checkbox>
              <el-checkbox value="weekday_afternoon">工作日下午</el-checkbox>
              <el-checkbox value="weekday_evening">工作日晚上</el-checkbox>
              <el-checkbox value="weekend_morning">周末上午</el-checkbox>
              <el-checkbox value="weekend_afternoon">周末下午</el-checkbox>
              <el-checkbox value="weekend_evening">周末晚上</el-checkbox>
            </el-checkbox-group>
          </div>
        </div>
      </el-collapse-transition>

      <div v-if="activeFilterTags.length" class="active-tags-bar">
        <span class="active-tags-label">当前筛选：</span>
        <el-tag
          v-for="(tag, idx) in activeFilterTags"
          :key="idx"
          :type="tag.type || 'info'"
          size="small"
          closable
          @close="removeFilter(tag.key)"
          class="active-tag"
        >
          {{ tag.label }}
        </el-tag>
        <el-link type="primary" :underline="false" @click="resetFilters" class="clear-all-link">全部清除</el-link>
      </div>

      <div v-if="users.length" class="users-grid">
        <div v-for="user in users" :key="user.id" class="user-card" @click="goToProfile(user.id)">
          <el-avatar :src="user.avatar" :size="64" />
          <div class="user-name">{{ user.username }}</div>
          <div class="user-rating">
            <el-rate :model-value="user.rating" disabled />
            <span class="rating-text">{{ user.rating }}</span>
          </div>
          <div class="user-stats">
            <span>交换 {{ user.exchangeCount || 0 }} 次</span>
            <span v-if="user.preferences?.location?.city" class="location-tag">
              <el-icon><Location /></el-icon>{{ user.preferences.location.city }}
            </span>
          </div>
          <div class="user-skill-tags">
            <span v-for="s in (user.teachSkills || []).slice(0, 2)" :key="s.id" class="skill-tag skill-teach" :title="s.name">
              🎓 {{ s.name.length > 6 ? s.name.slice(0, 6) + '...' : s.name }}
            </span>
            <span v-for="s in (user.learnSkills || []).slice(0, 2)" :key="s.id" class="skill-tag skill-learn" :title="s.name">
              📖 {{ s.name.length > 6 ? s.name.slice(0, 6) + '...' : s.name }}
            </span>
          </div>
          <div class="user-bio">{{ user.bio || '这个人很懒，什么都没写' }}</div>
          <div class="user-card-actions" @click.stop>
            <el-button type="primary" size="small" plain @click="goToChat(user.id)">
              <el-icon><ChatDotRound /></el-icon>聊一聊
            </el-button>
          </div>
        </div>
      </div>
      <el-empty v-else description="没有找到符合条件的用户，试试调整筛选条件？">
        <template #action>
          <el-button type="primary" @click="resetFilters()">清除筛选</el-button>
        </template>
      </el-empty>
    </div>

    <el-dialog v-model="saveDialogVisible" title="保存筛选条件" width="460px">
      <el-form label-width="90px">
        <el-form-item label="条件名称">
          <el-input
            v-model="newFilterName"
            placeholder="给当前筛选条件起个名字"
            maxlength="50"
            show-word-limit
          />
        </el-form-item>
        <el-form-item label="筛选摘要">
          <div class="filter-summary">
            <el-tag
              v-for="(tag, idx) in activeFilterTags.slice(0, 8)"
              :key="idx"
              :type="tag.type || 'info'"
              size="small"
              class="summary-tag"
            >
              {{ tag.label }}
            </el-tag>
            <span v-if="activeFilterTags.length > 8" class="more-tags">
              等 {{ activeFilterTags.length }} 个条件
            </span>
            <span v-if="!activeFilterTags.length" class="empty-summary">
              （当前没有设置任何筛选条件）
            </span>
          </div>
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="saveDialogVisible = false">取消</el-button>
        <el-button type="primary" :disabled="!newFilterName || !activeFilterTags.length" @click="handleSaveFilter">
          保存
        </el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, reactive } from 'vue'
import { useRouter } from 'vue-router'
import { ElMessage, ElMessageBox } from 'element-plus'
import { statsAPI, matchAPI, authAPI, skillAPI, filterAPI } from '../api'
import {
  Search, ArrowUp, ArrowDown, Star, Refresh, Filter,
  Collection, Location, ChatDotRound
} from '@element-plus/icons-vue'

const router = useRouter()
const stats = ref({ totalUsers: 0, totalSkills: 0, completedExchanges: 0, successRate: 0 })
const popularSkills = ref([])
const recommendedMatches = ref([])
const users = ref([])
const categories = ref([])
const cityList = ref([])
const provinceList = ref([])
const savedFilters = ref([])

const showAdvancedFilter = ref(false)
const showSavedFilters = ref(false)
const saveDialogVisible = ref(false)
const newFilterName = ref('')

const filters = reactive({
  keyword: '',
  city: '',
  province: '',
  minRating: '',
  minExchangeCount: '',
  teachSkill: '',
  teachSkillCategory: '',
  learnSkill: '',
  learnSkillCategory: '',
  exchangeMode: 'both',
  availableTime: []
})

const TIME_LABELS = {
  weekday_morning: '工作日上午',
  weekday_afternoon: '工作日下午',
  weekday_evening: '工作日晚上',
  weekend_morning: '周末上午',
  weekend_afternoon: '周末下午',
  weekend_evening: '周末晚上'
}

const activeFilterTags = computed(() => {
  const tags = []
  if (filters.keyword) tags.push({ key: 'keyword', label: `关键词: ${filters.keyword}`, type: 'primary' })
  if (filters.province) tags.push({ key: 'province', label: `省份: ${filters.province}`, type: '' })
  if (filters.city) tags.push({ key: 'city', label: `城市: ${filters.city}`, type: '' })
  if (filters.minRating) tags.push({ key: 'minRating', label: `评分≥${filters.minRating}`, type: 'warning' })
  if (filters.minExchangeCount) tags.push({ key: 'minExchangeCount', label: `交换≥${filters.minExchangeCount}次`, type: 'warning' })
  if (filters.teachSkillCategory) {
    const cat = categories.value.find(c => c.id === filters.teachSkillCategory)
    tags.push({ key: 'teachSkillCategory', label: `对方可教类别: ${cat ? cat.name : filters.teachSkillCategory}`, type: 'success' })
  }
  if (filters.teachSkill) tags.push({ key: 'teachSkill', label: `对方可教: ${filters.teachSkill}`, type: 'success' })
  if (filters.learnSkillCategory) {
    const cat = categories.value.find(c => c.id === filters.learnSkillCategory)
    tags.push({ key: 'learnSkillCategory', label: `对方想学类别: ${cat ? cat.name : filters.learnSkillCategory}`, type: 'danger' })
  }
  if (filters.learnSkill) tags.push({ key: 'learnSkill', label: `对方想学: ${filters.learnSkill}`, type: 'danger' })
  if (filters.exchangeMode && filters.exchangeMode !== 'both') {
    tags.push({ key: 'exchangeMode', label: `交换方式: ${filters.exchangeMode === 'online' ? '线上' : '线下'}`, type: 'info' })
  }
  if (filters.availableTime && filters.availableTime.length) {
    const times = filters.availableTime.slice(0, 2).map(t => TIME_LABELS[t] || t).join('、')
    const more = filters.availableTime.length > 2 ? `等${filters.availableTime.length}个` : ''
    tags.push({ key: 'availableTime', label: `可用时间: ${times}${more}`, type: 'info' })
  }
  return tags
})

onMounted(async () => {
  await Promise.all([
    loadStats(),
    loadPopularSkills(),
    loadMatches(),
    loadCategories(),
    loadCities(),
    loadSavedFilters(),
    loadUsers()
  ])
})

async function loadStats() {
  const res = await statsAPI.getSuccessRate()
  stats.value = res.data
}

async function loadPopularSkills() {
  const res = await statsAPI.getPopularSkills()
  popularSkills.value = res.data.slice(0, 10)
}

async function loadMatches() {
  try {
    const res = await matchAPI.getMatches()
    recommendedMatches.value = res.data
  } catch (e) {}
}

async function loadCategories() {
  const res = await skillAPI.getCategories()
  categories.value = res.data
}

async function loadCities() {
  try {
    const res = await authAPI.getCities()
    cityList.value = res.data.cities
    provinceList.value = res.data.provinces
  } catch (e) {}
}

async function loadSavedFilters() {
  try {
    const res = await filterAPI.getFilters()
    savedFilters.value = res.data
  } catch (e) {}
}

async function loadUsers() {
  const params = {}
  if (filters.keyword) params.keyword = filters.keyword
  if (filters.city) params.city = filters.city
  if (filters.province) params.province = filters.province
  if (filters.minRating) params.minRating = filters.minRating
  if (filters.minExchangeCount) params.minExchangeCount = filters.minExchangeCount
  if (filters.teachSkill) params.teachSkill = filters.teachSkill
  if (filters.teachSkillCategory) params.teachSkillCategory = filters.teachSkillCategory
  if (filters.learnSkill) params.learnSkill = filters.learnSkill
  if (filters.learnSkillCategory) params.learnSkillCategory = filters.learnSkillCategory
  if (filters.exchangeMode && filters.exchangeMode !== 'both') params.exchangeMode = filters.exchangeMode
  if (filters.availableTime && filters.availableTime.length) params.availableTime = filters.availableTime.join(',')

  const res = await authAPI.getUsers(params)
  users.value = res.data.slice(0, 16)
}

function toggleAdvancedFilter() {
  showAdvancedFilter.value = !showAdvancedFilter.value
}

async function resetFilters() {
  filters.keyword = ''
  filters.city = ''
  filters.province = ''
  filters.minRating = ''
  filters.minExchangeCount = ''
  filters.teachSkill = ''
  filters.teachSkillCategory = ''
  filters.learnSkill = ''
  filters.learnSkillCategory = ''
  filters.exchangeMode = 'both'
  filters.availableTime = []
  await loadUsers()
}

async function removeFilter(key) {
  switch (key) {
    case 'keyword': filters.keyword = ''; break
    case 'city': filters.city = ''; break
    case 'province': filters.province = ''; break
    case 'minRating': filters.minRating = ''; break
    case 'minExchangeCount': filters.minExchangeCount = ''; break
    case 'teachSkill': filters.teachSkill = ''; break
    case 'teachSkillCategory': filters.teachSkillCategory = ''; break
    case 'learnSkill': filters.learnSkill = ''; break
    case 'learnSkillCategory': filters.learnSkillCategory = ''; break
    case 'exchangeMode': filters.exchangeMode = 'both'; break
    case 'availableTime': filters.availableTime = []; break
  }
  await loadUsers()
}

function openSaveDialog() {
  if (!activeFilterTags.value.length) {
    ElMessage.warning('请先设置筛选条件再保存')
    return
  }
  newFilterName.value = ''
  saveDialogVisible.value = true
}

async function handleSaveFilter() {
  if (!newFilterName.value.trim()) {
    ElMessage.warning('请输入条件名称')
    return
  }
  const snapshot = JSON.parse(JSON.stringify(filters))
  try {
    await filterAPI.saveFilter({
      name: newFilterName.value.trim(),
      filters: snapshot
    })
    ElMessage.success('筛选条件已保存')
    saveDialogVisible.value = false
    await loadSavedFilters()
    showSavedFilters.value = true
  } catch (e) {
    ElMessage.error(e.message || '保存失败')
  }
}

function applySavedFilter(saved) {
  const f = saved.filters || {}
  filters.keyword = f.keyword || ''
  filters.city = f.city || ''
  filters.province = f.province || ''
  filters.minRating = f.minRating || ''
  filters.minExchangeCount = f.minExchangeCount || ''
  filters.teachSkill = f.teachSkill || ''
  filters.teachSkillCategory = f.teachSkillCategory || ''
  filters.learnSkill = f.learnSkill || ''
  filters.learnSkillCategory = f.learnSkillCategory || ''
  filters.exchangeMode = f.exchangeMode || 'both'
  filters.availableTime = Array.isArray(f.availableTime) ? [...f.availableTime] : []
  loadUsers()
  ElMessage.success(`已恢复筛选条件：${saved.name}`)
}

async function handleDeleteFilter(saved) {
  try {
    await ElMessageBox.confirm(
      `确定要删除筛选条件"${saved.name}"吗？`,
      '删除确认',
      { type: 'warning', confirmButtonText: '删除', cancelButtonText: '取消' }
    )
    await filterAPI.deleteFilter(saved.id)
    await loadSavedFilters()
    ElMessage.success('已删除')
  } catch (e) {
    if (e !== 'cancel') {
      ElMessage.error('删除失败')
    }
  }
}

function getScoreClass(score) {
  if (score >= 70) return 'score-high'
  if (score >= 40) return 'score-medium'
  return 'score-low'
}

function goToChat(userId) {
  router.push(`/chat/${userId}`)
}

function goToProfile(userId) {
  router.push(`/profile/${userId}`)
}
</script>

<style scoped>
.home {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.stats-row {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 16px;
}

.stat-card {
  background: white;
  border-radius: 12px;
  padding: 24px;
  display: flex;
  align-items: center;
  gap: 16px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
}

.stat-icon {
  font-size: 48px;
}

.stat-value {
  font-size: 28px;
  font-weight: 700;
  color: #333;
}

.stat-label {
  font-size: 14px;
  color: #999;
}

.main-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 24px;
}

.skill-rank-item {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 12px 0;
  border-bottom: 1px solid #eee;
}

.rank-number {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: 14px;
  background: #f0f0f0;
  color: #666;
}

.rank-1 {
  background: linear-gradient(135deg, #ffd700, #ffb700);
  color: white;
}

.rank-2 {
  background: linear-gradient(135deg, #c0c0c0, #a0a0a0);
  color: white;
}

.rank-3 {
  background: linear-gradient(135deg, #cd7f32, #b87333);
  color: white;
}

.skill-info {
  flex: 1;
}

.skill-name {
  font-weight: 600;
  color: #333;
}

.count-tag {
  padding: 2px 8px;
  border-radius: 10px;
  font-size: 12px;
  margin-right: 8px;
}

.count-tag.teach {
  background: #e8f5e9;
  color: #2e7d32;
}

.count-tag.learn {
  background: #fff3e0;
  color: #ef6c00;
}

.skill-demand {
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 500;
}

.high-demand {
  background: #ffebee;
  color: #c62828;
}

.low-demand {
  background: #e8f5e9;
  color: #2e7d32;
}

.match-card {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px 0;
  border-bottom: 1px solid #eee;
}

.match-info {
  flex: 1;
}

.match-header {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 8px;
}

.match-name {
  font-weight: 600;
  color: #333;
}

.match-skills {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.discover-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 16px;
}

.section-title {
  font-size: 18px;
  font-weight: 600;
  color: #333;
  margin-bottom: 16px;
}

.header-actions {
  display: flex;
  gap: 8px;
}

.saved-filters-bar {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  background: linear-gradient(135deg, #f0f9ff 0%, #e0f2fe 100%);
  border-radius: 10px;
  margin-bottom: 16px;
  border: 1px solid #bae6fd;
}

.saved-label {
  font-size: 13px;
  font-weight: 500;
  color: #0369a1;
  white-space: nowrap;
}

.saved-tags {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  flex: 1;
}

.saved-filter-tag {
  cursor: pointer;
  transition: all 0.2s;
}

.saved-filter-tag:hover {
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(34, 197, 94, 0.3);
}

.basic-filter-bar {
  display: flex;
  gap: 12px;
  margin-bottom: 16px;
  flex-wrap: wrap;
  align-items: center;
}

.advanced-filter-panel {
  background: #fafbfc;
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 16px;
  border: 1px solid #e5e7eb;
}

.filter-section {
  padding: 14px 0;
  border-bottom: 1px dashed #e5e7eb;
}

.filter-section:last-child {
  border-bottom: none;
  padding-bottom: 0;
}

.filter-section:first-child {
  padding-top: 0;
}

.filter-section-title {
  font-size: 14px;
  font-weight: 600;
  color: #374151;
  margin-bottom: 12px;
  display: flex;
  align-items: center;
  gap: 6px;
}

.filter-row {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  align-items: center;
}

.filter-block {
  padding: 10px 0;
}

.filter-block-label {
  font-size: 13px;
  color: #6b7280;
  margin-bottom: 8px;
  display: flex;
  align-items: center;
  gap: 6px;
}

.block-icon {
  font-size: 14px;
}

.active-tags-bar {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
  padding: 12px 16px;
  background: #fff7ed;
  border-radius: 10px;
  margin-bottom: 20px;
  border: 1px solid #fed7aa;
}

.active-tags-label {
  font-size: 13px;
  font-weight: 500;
  color: #c2410c;
  white-space: nowrap;
}

.active-tag {
  cursor: default;
}

.clear-all-link {
  margin-left: auto;
  font-size: 13px;
  cursor: pointer;
}

.users-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 16px;
}

.user-card {
  background: #fafafa;
  border-radius: 12px;
  padding: 20px 16px;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s;
  border: 2px solid transparent;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 6px;
}

.user-card:hover {
  border-color: #667eea;
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(102, 126, 234, 0.2);
  background: #fff;
}

.user-name {
  font-weight: 600;
  color: #333;
  margin-top: 8px;
  font-size: 15px;
}

.user-rating {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
}

.rating-text {
  font-weight: 600;
  color: #ff9800;
  font-size: 13px;
}

.user-stats {
  font-size: 12px;
  color: #666;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  flex-wrap: wrap;
}

.location-tag {
  display: inline-flex;
  align-items: center;
  gap: 2px;
  color: #6366f1;
}

.user-skill-tags {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  gap: 4px;
  margin: 6px 0;
  min-height: 24px;
}

.skill-tag {
  display: inline-block;
  padding: 2px 8px;
  border-radius: 6px;
  font-size: 11px;
  font-weight: 500;
  max-width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.skill-teach {
  background: #dcfce7;
  color: #15803d;
}

.skill-learn {
  background: #fef3c7;
  color: #b45309;
}

.user-bio {
  font-size: 12px;
  color: #999;
  line-height: 1.5;
  max-width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  min-height: 36px;
}

.user-card-actions {
  margin-top: 8px;
}

.filter-summary {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  padding: 12px;
  background: #f9fafb;
  border-radius: 8px;
  min-height: 48px;
}

.summary-tag {
  margin: 0;
}

.more-tags {
  font-size: 12px;
  color: #6b7280;
  align-self: center;
}

.empty-summary {
  font-size: 13px;
  color: #9ca3af;
  font-style: italic;
}

.score-badge {
  padding: 2px 10px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
}

.score-high {
  background: linear-gradient(135deg, #10b981, #059669);
  color: white;
}

.score-medium {
  background: linear-gradient(135deg, #f59e0b, #d97706);
  color: white;
}

.score-low {
  background: linear-gradient(135deg, #9ca3af, #6b7280);
  color: white;
}

@media (max-width: 1024px) {
  .stats-row {
    grid-template-columns: repeat(2, 1fr);
  }
  .main-grid {
    grid-template-columns: 1fr;
  }
  .users-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 640px) {
  .stats-row {
    grid-template-columns: 1fr;
  }
  .users-grid {
    grid-template-columns: 1fr;
  }
}
</style>
