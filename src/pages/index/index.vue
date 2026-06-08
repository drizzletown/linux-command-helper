<template>
  <view class="container">
    <!-- 1. 智能搜索框 (始终置顶) -->
    <view class="search-box">
      <input 
        v-model="searchQuery" 
        placeholder="输入命令或描述 (如: 查找文件, gre...)" 
        class="search-input"
        @input="handleSearch"
      />
    </view>

    <!-- 2. 顶部 Tab 切换栏 -->
    <view class="tab-header">
      <view 
        class="tab-item" 
        :class="{ 'active': activeTab === 'random' }"
        @click="switchTab('random')"
      >
        <text class="tab-text">🎲 随机发现</text>
      </view>
      <view 
        class="tab-item" 
        :class="{ 'active': activeTab === 'category' }"
        @click="switchTab('category')"
      >
        <text class="tab-text">📂 分类浏览</text>
      </view>
    </view>

    <!-- 3. 内容展示区 -->
    <view class="content-area">
      
      <!-- 状态 A: 搜索有结果时，优先展示搜索结果 -->
      <scroll-view scroll-y class="result-list" v-if="searchQuery && searchResults.length > 0">
        <view v-for="item in searchResults" :key="item.id" class="result-item" @click="goToDetail(item.id)">
          <text class="cmd-name">{{ item.name }}</text>
          <text class="cmd-desc">{{ item.shortDesc }}</text>
        </view>
      </scroll-view>
      
      <!-- 状态 B: 搜索无结果提示 -->
      <view v-else-if="searchQuery && searchResults.length === 0" class="no-result">
        未找到相关命令，试试换个关键词？
      </view>

      <!-- 状态 C: 数据加载中 -->
      <view v-else-if="!isDataLoaded" class="loading-state">
        <text class="loading-text">正在加载命令数据...</text>
      </view>

      <!-- 状态 D: 无搜索时，根据 Tab 显示不同内容 -->
      <view v-else>
        
        <!-- Tab 1: 随机发现 -->
        <view v-if="activeTab === 'random'" class="random-section">
          <!-- 点击整个卡片触发随机切换 -->
          <view 
            class="random-card" 
            :class="{ 'card-flipping': isFlipping }" 
            @click="handleRefresh"
          >
            <view class="card-header">
              <view class="name-wrapper">
                <text class="cmd-name">{{ randomCmd.name }}</text>
                <text class="category-tag">{{ randomCmd.category }}</text>
              </view>
            </view>
            <text class="card-desc">{{ randomCmd.shortDesc }}</text>

            <view class="card-footer">
              <view class="related-tags" v-if="randomCmd.related && randomCmd.related.length">
                <text class="related-label">关联：</text>
                <text class="related-item" v-for="(rel, idx) in randomCmd.related.slice(0, 3)" :key="idx">
                  {{ rel }}{{ idx < Math.min(randomCmd.related.length, 3) - 1 ? '、' : '' }}
                </text>
              </view>
              
              <!-- 右下角"查看详情"按钮 -->
              <view class="detail-btn" @click.stop="goToDetail(randomCmd.id)">
                <text class="detail-text">查看详情</text>
                <text class="detail-icon">›</text>
              </view>
            </view>
          </view>
          
          <!-- 底部提示语 -->
          <view class="tap-hint">
            <text>💡 点击卡片空白处可随机切换命令</text>
          </view>
        </view>

        <!-- Tab 2: 分类浏览 -->
        <view v-if="activeTab === 'category'" class="category-section">
          <view class="category-grid">
            <view 
              v-for="cat in categories" 
              :key="cat.name" 
              class="category-item"
              @click="goToCategory(cat.name)"
            >
              <text class="cat-icon">{{ cat.icon }}</text>
              <text class="cat-name">{{ cat.name }}</text>
            </view>
          </view>
        </view>

      </view>
    </view>
  </view>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import Fuse from 'fuse.js'

const searchQuery = ref('')
const searchResults = ref([])
const randomCmd = ref({})
const isFlipping = ref(false)
const activeTab = ref('random')
const isDataLoaded = ref(false)
const commandsData = ref([])
const fuse = ref(null)
let searchTimer = null

const categories = [
  { name: '文件管理', icon: '📁' },
  { name: '文本处理', icon: '' },
  { name: '系统管理', icon: '⚙️' },
  { name: '网络', icon: '🌐' },
  { name: '打包压缩', icon: '📦' },
  { name: '开发', icon: '💻' }
]

// 加载 JSON 数据（兼容本地开发和 GitHub Pages）
const loadData = async () => {
  try {
    try {
      const module = await import('@/data/linux_commands.json')
      commandsData.value = module.default || module
    } catch (e) {
      const response = await fetch('./linux_commands.json')
      if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`)
      commandsData.value = await response.json()
    }
    
    fuse.value = new Fuse(commandsData.value, {
      keys: [
        { name: 'name', weight: 0.6 },
        { name: 'keywords', weight: 0.3 },
        { name: 'shortDesc', weight: 0.1 }
      ],
      threshold: 0.3,
      includeScore: true
    })
    
    isDataLoaded.value = true
    getRandomCommand()
    
    console.log('数据加载成功，共', commandsData.value.length, '条命令')
  } catch (error) {
    console.error('加载命令数据失败:', error)
    uni.showToast({ 
      title: '数据加载失败，请刷新页面', 
      icon: 'none',
      duration: 3000 
    })
  }
}

const handleSearch = () => {
  if (!fuse.value) return
  if (searchTimer) clearTimeout(searchTimer)
  searchTimer = setTimeout(() => {
    if (!searchQuery.value.trim()) {
      searchResults.value = []
      return
    }
    const results = fuse.value.search(searchQuery.value)
    searchResults.value = results.map(r => r.item).slice(0, 10)
  }, 300)
}

const getRandomCommand = () => {
  if (commandsData.value.length === 0) return
  let newIndex
  do {
    newIndex = Math.floor(Math.random() * commandsData.value.length)
  } while (commandsData.value[newIndex].id === randomCmd.value.id && commandsData.value.length > 1)
  randomCmd.value = commandsData.value[newIndex]
}

const handleRefresh = () => {
  if (isFlipping.value) return
  isFlipping.value = true
  uni.vibrateShort({ type: 'light' })
  setTimeout(() => { getRandomCommand() }, 200)
  setTimeout(() => { isFlipping.value = false }, 400)
}

const goToDetail = (id) => {
  uni.navigateTo({ url: `/pages/detail/detail?id=${id}` })
}

const goToCategory = (categoryName) => {
  uni.navigateTo({ url: `/pages/category/category?name=${encodeURIComponent(categoryName)}` })
}

const switchTab = (tab) => {
  activeTab.value = tab
  searchQuery.value = ''
  searchResults.value = []
}

onMounted(() => {
  loadData()
})
</script>

<style scoped>
/* 基础容器 */
.container { 
  padding: 15px; 
  background-color: var(--bg-color); 
  min-height: 100vh; 
  padding-bottom: 40px; 
  transition: background-color 0.3s;
}

/* 搜索框 */
.search-box { margin-bottom: 15px; }
.search-input { 
  background: var(--card-bg); 
  color: var(--text-primary);
  padding: 12px 15px; 
  border-radius: 25px; 
  font-size: 15px; 
  box-shadow: 0 2px 8px rgba(0,0,0,0.05); 
  transition: background-color 0.3s, color 0.3s;
}

/* 顶部 Tab 栏 */
.tab-header { 
  display: flex; 
  background: var(--card-bg); 
  padding: 6px; 
  border-radius: 12px;
  margin-bottom: 20px; 
  box-shadow: 0 2px 8px rgba(0,0,0,0.04);
  transition: background-color 0.3s;
}
.tab-item { 
  flex: 1; 
  text-align: center; 
  padding: 10px 0; 
  border-radius: 8px; 
  transition: all 0.3s; 
}
.tab-item.active { 
  background: #e6f7ff; 
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}
.tab-text { 
  font-size: 14px; 
  color: var(--text-secondary); 
  font-weight: 500; 
}
.tab-item.active .tab-text { 
  color: #007AFF; 
  font-weight: bold; 
}

.content-area { min-height: 300px; }

/* 搜索结果 */
.result-list { 
  max-height: 400px; 
  background: var(--card-bg); 
  border-radius: 12px; 
  box-shadow: 0 4px 12px rgba(0,0,0,0.08); 
  transition: background-color 0.3s;
}
.result-item { 
  padding: 15px; 
  border-bottom: 1px solid var(--border-color); 
  display: flex; 
  flex-direction: column; 
  transition: border-color 0.3s;
}
.result-item:last-child { border-bottom: none; }
.cmd-name { font-weight: bold; color: var(--theme-color); font-family: monospace; font-size: 16px; }
.cmd-desc { font-size: 13px; color: var(--text-secondary); margin-top: 4px; }
.no-result { padding: 40px 20px; text-align: center; color: var(--text-tertiary); font-size: 14px; }

/* 加载状态 */
.loading-state {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 60px 20px;
}
.loading-text {
  color: var(--text-secondary);
  font-size: 14px;
}

/* --- 随机卡片样式（精简版）--- */
.random-section { padding: 0 5px; }
.random-card { 
  background: linear-gradient(135deg, #E0F2FE 0%, #DBEAFE 100%); 
  color: #0F172A; 
  padding: 24px 20px; 
  border-radius: 16px; 
  box-shadow: 0 4px 15px rgba(186, 230, 253, 0.6); 
  position: relative;
  overflow: hidden;
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
  border: 1px solid rgba(255, 255, 255, 0.6); 
  cursor: pointer;
}
.random-card:active { transform: scale(0.98); box-shadow: 0 2px 8px rgba(186, 230, 253, 0.4); }
.random-card.card-flipping { transform: scale(0.95) rotateX(5deg); opacity: 0.9; }

.card-header { margin-bottom: 12px; }
.name-wrapper { display: flex; align-items: center; gap: 10px; }
.cmd-name { 
  font-size: 36px; 
  font-weight: 900; 
  font-family: monospace; 
  letter-spacing: 1px; 
  color: #0369A1; 
  text-shadow: none; 
}
.category-tag { 
  background: rgba(255, 255, 255, 0.7); 
  color: #0284C7;
  padding: 4px 10px; 
  border-radius: 12px; 
  font-size: 12px; 
  font-weight: bold;
  backdrop-filter: blur(4px); 
  border: 1px solid rgba(255, 255, 255, 0.9);
}
.card-desc { 
  font-size: 17px; 
  color: #334155; 
  line-height: 1.6; 
  display: block; 
  margin-bottom: 20px; 
  font-weight: 500;
}

.card-footer { display: flex; justify-content: space-between; align-items: flex-end; }
.related-tags { flex: 1; font-size: 13px; color: #475569; line-height: 1.4; font-weight: 500; }
.related-label { color: #64748B; }
.related-item { color: #0369A1; font-weight: bold; }

/* 查看详情按钮样式 */
.detail-btn { 
  display: flex; 
  align-items: center; 
  background: #0284C7;
  color: #ffffff;
  padding: 6px 14px; 
  border-radius: 20px; 
  margin-left: 10px;
  box-shadow: 0 2px 5px rgba(2, 132, 199, 0.3);
}
.detail-btn:active { transform: scale(0.95); background: #0369A1; }
.detail-text { font-size: 13px; font-weight: bold; }
.detail-icon { font-size: 16px; margin-left: 2px; font-weight: bold; line-height: 1; }

/* 底部提示语 */
.tap-hint {
  text-align: center;
  margin-top: 15px;
  font-size: 12px;
  color: var(--text-tertiary);
}

/* 分类网格 */
.category-section { padding: 0 5px; }
.category-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 15px; }
.category-item { 
  background: var(--card-bg); 
  border-radius: 12px; 
  padding: 20px 10px; 
  display: flex; 
  flex-direction: column; 
  align-items: center; 
  justify-content: center; 
  box-shadow: 0 2px 8px rgba(0,0,0,0.04); 
  transition: transform 0.1s, background-color 0.3s; 
}
.category-item:active { transform: scale(0.95); background: var(--border-color); }
.cat-icon { font-size: 32px; margin-bottom: 10px; }
.cat-name { font-size: 14px; color: var(--text-primary); font-weight: 500; }

/* 暗黑模式适配 */
@media (prefers-color-scheme: dark) {
  .tab-item.active { background: rgba(10, 132, 255, 0.2); }
  .random-card {
    background: linear-gradient(135deg, #1E293B 0%, #0F172A 100%); 
    color: #F8FAFC;
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
    border-color: rgba(255, 255, 255, 0.1);
  }
  .cmd-name { color: #38BDF8; }
  .category-tag { background: rgba(255, 255, 255, 0.1); color: #7DD3FC; border-color: rgba(255,255,255,0.2); }
  .card-desc { color: #CBD5E1; }
  .related-tags { color: #94A3B8; }
  .related-label { color: #64748B; }
  .related-item { color: #38BDF8; }
  .detail-btn { background: #0284C7; color: #fff; }
}
</style>