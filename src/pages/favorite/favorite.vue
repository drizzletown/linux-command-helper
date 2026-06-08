<template>
  <view class="container">
    <!-- 顶部 Tab 切换 -->
    <view class="tab-header">
      <view 
        class="tab-item" 
        :class="{ 'active': activeTab === 'favorites' }"
        @click="switchTab('favorites')"
      >
        <text class="tab-text">我的收藏 ({{ favorites.length }})</text>
      </view>
      <view 
        class="tab-item" 
        :class="{ 'active': activeTab === 'wrongs' }"
        @click="switchTab('wrongs')"
      >
        <text class="tab-text">错题本 ({{ wrongAnswers.length }})</text>
      </view>
    </view>

    <!-- 列表内容区 -->
    <scroll-view scroll-y class="list-container">
      
      <!-- 状态 1: 我的收藏 -->
      <view v-if="activeTab === 'favorites'">
        <view v-if="favorites.length === 0" class="empty-state">
          <text class="empty-icon">📚</text>
          <text class="empty-text">还没有收藏任何命令</text>
          <text class="empty-desc">在详情页点击 ❤️ 即可收藏</text>
        </view>
        
        <view v-for="cmd in favoriteCommands" :key="cmd.id" class="command-card">
          <view class="card-content" @click="goToDetail(cmd.id)">
            <view class="cmd-header">
              <text class="cmd-name">{{ cmd.name }}</text>
              <text class="category-tag">{{ cmd.category }}</text>
            </view>
            <text class="cmd-desc">{{ cmd.shortDesc }}</text>
          </view>
          <view class="card-action" @click="removeFavorite(cmd.id)">
            <text class="action-icon">❤️</text>
          </view>
        </view>
      </view>

      <!-- 状态 2: 错题本 -->
      <view v-if="activeTab === 'wrongs'">
        <view v-if="wrongAnswers.length === 0" class="empty-state">
          <text class="empty-icon">🎉</text>
          <text class="empty-text">太棒了！错题本是空的</text>
          <text class="empty-desc">去“每日一练”挑战一下吧</text>
        </view>
        
        <view v-for="cmd in wrongCommands" :key="cmd.id" class="command-card wrong-card">
          <view class="card-content" @click="goToDetail(cmd.id)">
            <view class="cmd-header">
              <text class="cmd-name">{{ cmd.name }}</text>
              <text class="category-tag">{{ cmd.category }}</text>
            </view>
            <text class="cmd-desc">{{ cmd.shortDesc }}</text>
          </view>
          <view class="card-action" @click="removeWrong(cmd.id)">
            <text class="action-icon">✅</text>
            <text class="action-text">已掌握</text>
          </view>
        </view>
      </view>

    </scroll-view>
  </view>
</template>

<script setup>
import { ref, computed } from 'vue'
import { onShow } from '@dcloudio/uni-app'

const activeTab = ref('favorites')
const favorites = ref([])
const wrongAnswers = ref([])

// 核心修改：改为响应式数组，用于存放动态加载的数据
const commandsData = ref([])

// 核心修改：动态加载 JSON 数据
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
    console.log('学习中心数据加载成功，共', commandsData.value.length, '条命令')
  } catch (error) {
    console.error('加载命令数据失败:', error)
    uni.showToast({ title: '数据加载失败', icon: 'none' })
  }
}

// 核心：根据 ID 数组从总库中过滤出完整的命令对象
const favoriteCommands = computed(() => {
  return commandsData.value.filter(cmd => favorites.value.includes(cmd.id))
})

const wrongCommands = computed(() => {
  return commandsData.value.filter(cmd => wrongAnswers.value.includes(cmd.id))
})

// 加载本地存储的 ID 列表
const loadLocalData = () => {
  favorites.value = JSON.parse(uni.getStorageSync('linux_favorites') || '[]')
  wrongAnswers.value = JSON.parse(uni.getStorageSync('linux_wrong_answers') || '[]')
}

const switchTab = (tab) => {
  activeTab.value = tab
}

const removeFavorite = (id) => {
  uni.showModal({
    title: '取消收藏',
    content: '确定要取消收藏该命令吗？',
    success: (res) => {
      if (res.confirm) {
        favorites.value = favorites.value.filter(favId => favId !== id)
        uni.setStorageSync('linux_favorites', JSON.stringify(favorites.value))
        uni.showToast({ title: '已取消收藏', icon: 'none' })
      }
    }
  })
}

const removeWrong = (id) => {
  wrongAnswers.value = wrongAnswers.value.filter(wrongId => wrongId !== id)
  uni.setStorageSync('linux_wrong_answers', JSON.stringify(wrongAnswers.value))
  uni.showToast({ title: '太棒了，已掌握！', icon: 'success' })
}

const goToDetail = (id) => {
  uni.navigateTo({ url: `/pages/detail/detail?id=${id}` })
}

// 关键修复：使用 onShow 代替 onMounted，确保每次切换回该页面都刷新数据
onShow(async () => {
  // 1. 先加载题库数据
  await loadData()
  // 2. 再加载本地存储的 ID
  loadLocalData()
})
</script>

<style scoped>
.container { background-color: var(--bg-color); min-height: 100vh; display: flex; flex-direction: column; transition: background-color 0.3s; }

/* 顶部 Tab 栏 */
.tab-header { 
  display: flex; 
  background: var(--card-bg); 
  padding: 6px; 
  border-radius: 12px;
  margin: 15px 15px 10px 15px; 
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
  transition: color 0.3s;
}
.tab-item.active .tab-text { 
  color: #007AFF; 
  font-weight: bold; 
}

/* 列表容器 */
.list-container { flex: 1; padding: 0 15px 15px 15px; }

/* 空状态 */
.empty-state { 
  display: flex; 
  flex-direction: column; 
  align-items: center; 
  justify-content: center; 
  padding: 60px 20px; 
  text-align: center; 
}
.empty-icon { font-size: 48px; margin-bottom: 15px; }
.empty-text { font-size: 16px; color: var(--text-primary); font-weight: bold; margin-bottom: 8px; transition: color 0.3s;}
.empty-desc { font-size: 13px; color: var(--text-tertiary); transition: color 0.3s;}

/* 命令卡片 */
.command-card { 
  background: var(--card-bg); 
  border-radius: 12px; 
  padding: 15px; 
  margin-bottom: 12px; 
  display: flex; 
  justify-content: space-between; 
  align-items: center; 
  box-shadow: 0 2px 8px rgba(0,0,0,0.04); 
  transition: transform 0.1s, background-color 0.3s; 
}
.command-card:active { transform: scale(0.98); }
.wrong-card { border-left: 4px solid #ff4d4f; }

.card-content { flex: 1; }
.cmd-header { display: flex; align-items: center; gap: 10px; margin-bottom: 6px; }
.cmd-name { font-size: 18px; font-weight: bold; color: var(--theme-color); font-family: monospace; transition: color 0.3s;}
.category-tag { background: var(--border-color); color: var(--text-secondary); font-size: 11px; padding: 2px 8px; border-radius: 10px; transition: all 0.3s;}
.cmd-desc { font-size: 13px; color: var(--text-secondary); line-height: 1.4; transition: color 0.3s;}

/* 操作按钮 */
.card-action { 
  display: flex; 
  flex-direction: column; 
  align-items: center; 
  justify-content: center; 
  padding: 8px; 
  margin-left: 10px; 
}
.action-icon { font-size: 20px; }
.action-text { font-size: 11px; color: #52c41a; margin-top: 2px; font-weight: bold; }
</style>