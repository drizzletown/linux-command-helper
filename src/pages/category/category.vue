<template>
  <view class="container">
    <!-- 加载中提示 -->
    <view v-if="!isDataLoaded" class="loading-container">
      <text class="loading-text">正在加载分类数据...</text>
    </view>

    <!-- 正常内容区 -->
    <view v-else>
      <view class="header">
        <text class="title">{{ categoryName }} ({{ filteredCommands.length }})</text>
      </view>

      <scroll-view scroll-y class="list-container">
        <view 
          v-for="cmd in filteredCommands" 
          :key="cmd.id" 
          class="command-item"
          @click="goToDetail(cmd.id)"
        >
          <view class="cmd-left">
            <text class="cmd-name">{{ cmd.name }}</text>
            <text class="cmd-desc">{{ cmd.shortDesc }}</text>
          </view>
          <text class="arrow">›</text>
        </view>
        
        <view v-if="filteredCommands.length === 0" class="empty">
          该分类下暂无命令
        </view>
      </scroll-view>
    </view>
  </view>
</template>

<script setup>
import { ref, computed } from 'vue'
import { onLoad } from '@dcloudio/uni-app'

const categoryName = ref('')
const isDataLoaded = ref(false)

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
    isDataLoaded.value = true
    console.log('分类页数据加载成功，共', commandsData.value.length, '条命令')
  } catch (error) {
    console.error('加载命令数据失败:', error)
    uni.showToast({ title: '数据加载失败', icon: 'none' })
  }
}


// 核心：根据分类名称过滤命令
const filteredCommands = computed(() => {
  if (!categoryName.value) return []
  return commandsData.value.filter(cmd => cmd.category === categoryName.value)
})

onLoad(async (options) => {
  // 1. 先等待数据加载完成
  await loadData()
  
  // 2. 数据加载完成后，再处理页面参数
  if (options && options.name) {
    categoryName.value = decodeURIComponent(options.name)
    // 动态设置导航栏标题
    uni.setNavigationBarTitle({ title: categoryName.value })
  }
})

const goToDetail = (id) => {
  uni.navigateTo({ url: `/pages/detail/detail?id=${id}` })
}
</script>

<style scoped>
.container { background-color: var(--bg-color); min-height: 100vh; transition: background-color 0.3s; }
.loading-container { display: flex; justify-content: center; align-items: center; height: 100vh; color: var(--text-secondary); font-size: 14px; }
.loading-text { color: var(--text-secondary); }

.header { 
  background: var(--card-bg); 
  padding: 15px; 
  border-bottom: 1px solid var(--border-color); 
  position: sticky; 
  top: 0; 
  z-index: 10; 
  transition: all 0.3s; 
}
.title { font-size: 18px; font-weight: bold; color: var(--text-primary); transition: color 0.3s; }

.list-container { height: calc(100vh - 60px); }
.command-item { 
  background: var(--card-bg); 
  padding: 15px; 
  margin: 10px 15px; 
  border-radius: 10px; 
  display: flex; 
  justify-content: space-between; 
  align-items: center; 
  box-shadow: 0 2px 6px rgba(0,0,0,0.03); 
  transition: all 0.3s; 
}
.command-item:active { background: var(--border-color); transform: scale(0.98); }

.cmd-left { display: flex; flex-direction: column; flex: 1; }
.cmd-name { font-size: 16px; font-weight: bold; color: var(--theme-color); font-family: monospace; margin-bottom: 4px; transition: color 0.3s; }
.cmd-desc { font-size: 13px; color: var(--text-secondary); transition: color 0.3s; }
.arrow { font-size: 24px; color: var(--text-tertiary); font-weight: bold; transition: color 0.3s; }

.empty { text-align: center; color: var(--text-tertiary); margin-top: 50px; font-size: 14px; transition: color 0.3s; }
</style>