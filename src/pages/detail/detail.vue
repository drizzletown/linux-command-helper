<template>
  <view class="container" v-if="command">
    <!-- 1. 头部信息：命令名称、描述与收藏 -->
    <view class="header-card">
      <view class="header-top">
        <text class="cmd-name">{{ command.name }}</text>
        <view class="fav-btn" @click="toggleFavorite">
          <text class="fav-icon">{{ isFavorite ? '❤️' : '🤍' }}</text>
        </view>
      </view>
      <text class="cmd-desc">{{ command.shortDesc }}</text>
      <view class="cmd-category-tag">{{ command.category }}</view>
    </view>

    <!-- 2. 标准语法区 (带一键复制) -->
    <view class="section-card">
      <view class="section-title">📖 标准语法</view>
      <view class="code-block">
        <rich-text :nodes="highlightSyntax"></rich-text>
        <view class="copy-btn" @click="copyToClipboard(command.syntax)">
          <text class="copy-icon">📋</text>
        </view>
      </view>
    </view>

    <!-- 3. 核心参数详解 (折叠面板，默认收缩) -->
    <view class="section-card" v-if="command.parameters && command.parameters.length > 0">
      <view class="section-title collapsible-title" @click="toggleParams">
        <text>⚙️ 核心参数详解 ({{ command.parameters.length }})</text>
        <text class="arrow" :class="{ 'arrow-up': isParamsExpanded }">▼</text>
      </view>
      
      <view class="params-content" :class="{ 'expanded': isParamsExpanded }">
        <view class="param-list">
          <view v-for="(param, index) in command.parameters" :key="index" class="param-item">
            <view class="param-flag-box">
              <text class="param-flag">{{ param.flag }}</text>
            </view>
            <text class="param-desc">{{ param.desc }}</text>
          </view>
        </view>
      </view>
    </view>

    <!-- 4. 典型使用示例 (带标签、高亮与复制) -->
    <view class="section-card">
      <view class="section-title">💡 典型示例 ({{ command.examples.length }})</view>
      <view v-for="(example, index) in command.examples" :key="index" class="example-item">
        <!-- 场景标签 -->
        <view class="example-tag" v-if="example.tag">{{ example.tag }}</view>
        <text class="example-desc">{{ example.desc }}</text>
        <view class="code-block">
          <rich-text :nodes="getHighlightedCode(example.code, example.highlights)"></rich-text>
          <view class="copy-btn" @click="copyToClipboard(example.code)">
            <text class="copy-icon">📋</text>
          </view>
        </view>
      </view>
    </view>

    <!-- 5. 个人笔记区 -->
    <view class="section-card">
      <view class="section-title">📝 我的笔记</view>
      <textarea 
        class="note-textarea" 
        v-model="userNote" 
        placeholder="在这里记录你的学习心得或特殊用法..." 
        maxlength="500"
        @blur="saveNote"
      ></textarea>
      <text class="note-tip">失去焦点时自动保存</text>
    </view>
  </view>

  <!-- 加载中或找不到数据的提示 -->
  <view v-else class="loading-container">
    <text>加载中或命令不存在...</text>
  </view>
</template>

<script setup>
import { ref, computed } from 'vue'
import { onLoad } from '@dcloudio/uni-app'

const command = ref(null)
const isFavorite = ref(false)
const userNote = ref('')
const isParamsExpanded = ref(false)

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
  } catch (error) {
    console.error('加载命令数据失败:', error)
    uni.showToast({ title: '数据加载失败', icon: 'none' })
  }
}

// 接收首页传递的 id
onLoad(async (options) => {
  // 1. 先等待数据加载完成
  await loadData()
  
  // 2. 数据加载完成后，再根据 id 查找命令
  if (options && options.id) {
    const found = commandsData.value.find(cmd => cmd.id === options.id)
    if (found) {
      command.value = found
      loadFavoriteStatus(options.id)
      loadUserNote(options.id)
    }
  }
})

// --- 核心功能 1: 代码高亮逻辑 (彻底修复 HTML 标签被正则误伤问题) ---
const getHighlightedCode = (code, highlights) => {
  if (!code) return ''
  
  // 1. 转义 HTML 特殊字符，防止 XSS
  let safeCode = code.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
  
  // 2. 使用占位符策略：先在纯文本上标记关键字
  let html = safeCode
  const placeholders = []
  
  if (highlights && highlights.length > 0) {
    highlights.forEach((keyword, index) => {
      const placeholder = `___HL_${index}___`
      const safeKeyword = keyword.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
      const regex = new RegExp(`(${escapeRegExp(safeKeyword)})`, 'g')
      html = html.replace(regex, placeholder)
      placeholders.push({ placeholder, safeKeyword })
    })
  }
  
  // 3. 将占位符统一替换为带样式的 span 标签
  placeholders.forEach(({ placeholder, safeKeyword }) => {
    const span = `<span style="color: #ff7b72; font-weight: 700; background: rgba(255, 123, 114, 0.15); padding: 0 2px; border-radius: 3px;">${safeKeyword}</span>`
    html = html.replace(new RegExp(placeholder, 'g'), span)
  })
  
  // 4. 包裹外层 span，确保默认文字颜色在深色背景下可见
  return `<span style="color: #e6edf3; font-family: 'Menlo', 'Monaco', 'Courier New', monospace; font-size: 14px; line-height: 1.5; word-break: break-all;">${html}</span>`
}

const escapeRegExp = (string) => {
  return string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
}

const highlightSyntax = computed(() => {
  if (!command.value) return ''
  let html = command.value.syntax.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
  html = html.replace(/(\[.*?\])/g, '<span style="color: #79c0ff;">$1</span>')
  return `<span style="color: #e6edf3; font-family: 'Menlo', 'Monaco', 'Courier New', monospace; font-size: 14px; line-height: 1.5; word-break: break-all;">${html}</span>`
})

// --- 核心功能 2: 一键复制 ---
const copyToClipboard = (text) => {
  uni.setClipboardData({
    data: text,
    success: () => {
      uni.showToast({ title: '已复制到剪贴板', icon: 'success', duration: 1500 })
      uni.vibrateShort({ type: 'light' })
    },
    fail: () => {
      uni.showToast({ title: '复制失败', icon: 'none' })
    }
  })
}

// --- 核心功能 3: 折叠面板 ---
const toggleParams = () => {
  isParamsExpanded.value = !isParamsExpanded.value
}

// --- 核心功能 4: 收藏与笔记 (本地持久化) ---
const loadFavoriteStatus = (id) => {
  const favs = JSON.parse(uni.getStorageSync('linux_favorites') || '[]')
  isFavorite.value = favs.includes(id)
}

const toggleFavorite = () => {
  const id = command.value.id
  let favs = JSON.parse(uni.getStorageSync('linux_favorites') || '[]')
  
  if (isFavorite.value) {
    favs = favs.filter(favId => favId !== id)
    uni.showToast({ title: '已取消收藏', icon: 'none' })
  } else {
    favs.push(id)
    uni.showToast({ title: '收藏成功', icon: 'success' })
  }
  
  isFavorite.value = !isFavorite.value
  uni.setStorageSync('linux_favorites', JSON.stringify(favs))
}

const loadUserNote = (id) => {
  const notes = JSON.parse(uni.getStorageSync('linux_notes') || '{}')
  userNote.value = notes[id] || ''
}

const saveNote = () => {
  const id = command.value.id
  const notes = JSON.parse(uni.getStorageSync('linux_notes') || '{}')
  if (userNote.value.trim()) {
    notes[id] = userNote.value
  } else {
    delete notes[id]
  }
  uni.setStorageSync('linux_notes', JSON.stringify(notes))
  uni.showToast({ title: '笔记已保存', icon: 'success', duration: 1000 })
}
</script>

<style scoped>
.container { padding: 15px; background-color: var(--bg-color); min-height: 100vh; padding-bottom: 40px; transition: background-color 0.3s; }
.loading-container { display: flex; justify-content: center; align-items: center; height: 100vh; color: var(--text-tertiary); }

/* 头部卡片 */
.header-card { background: linear-gradient(135deg, #007AFF 0%, #0056b3 100%); color: white; padding: 20px; border-radius: 16px; margin-bottom: 15px; box-shadow: 0 4px 15px rgba(0, 122, 255, 0.3); }
.header-top { display: flex; justify-content: space-between; align-items: flex-start; }
.cmd-name { font-size: 32px; font-weight: bold; font-family: monospace; letter-spacing: 1px; }
.fav-btn { padding: 5px; }
.fav-icon { font-size: 24px; }
.cmd-desc { font-size: 15px; opacity: 0.9; margin-top: 10px; display: block; line-height: 1.4; }
.cmd-category-tag { display: inline-block; background: rgba(255,255,255,0.2); padding: 4px 12px; border-radius: 12px; font-size: 12px; margin-top: 12px; }

/* 通用区块卡片 */
.section-card { background: var(--card-bg); border-radius: 12px; padding: 15px; margin-bottom: 15px; box-shadow: 0 2px 8px rgba(0,0,0,0.04); transition: background-color 0.3s; }
.section-title { font-size: 16px; font-weight: bold; color: var(--text-primary); margin-bottom: 12px; display: flex; align-items: center; transition: color 0.3s; }
.collapsible-title { cursor: pointer; user-select: none; justify-content: space-between; }
.arrow { font-size: 12px; color: var(--text-tertiary); transition: transform 0.3s ease; }
.arrow-up { transform: rotate(180deg); }

/* 代码块样式 (深色背景) */
.code-block { display: flex; align-items: center; background: #282c34; padding: 12px; border-radius: 8px; position: relative; min-height: 44px; }
.code-block rich-text { flex: 1; }
.copy-btn { background: rgba(255,255,255,0.1); padding: 6px; border-radius: 6px; margin-left: 10px; display: flex; align-items: center; justify-content: center; }
.copy-btn:active { background: rgba(255,255,255,0.2); }
.copy-icon { font-size: 16px; }

/* 参数折叠面板 */
.params-content { max-height: 0; overflow: hidden; transition: max-height 0.4s ease-out; }
.params-content.expanded { max-height: 1000px; }
.param-list { border-left: 3px solid #007AFF; padding-left: 12px; }
.param-item { margin-bottom: 12px; display: flex; flex-direction: column; }
.param-flag-box { background: var(--border-color); padding: 4px 8px; border-radius: 4px; display: inline-block; margin-bottom: 4px; align-self: flex-start; transition: background-color 0.3s; }
.param-flag { font-family: monospace; color: #ff7b72; font-weight: bold; font-size: 13px; }
.param-desc { font-size: 13px; color: var(--text-secondary); line-height: 1.5; padding-left: 4px; transition: color 0.3s; }

/* 示例列表 */
.example-item { margin-bottom: 20px; padding-bottom: 15px; border-bottom: 1px dashed var(--border-color); transition: border-color 0.3s; }
.example-item:last-child { margin-bottom: 0; padding-bottom: 0; border-bottom: none; }
.example-tag { display: inline-block; background: #e6f7ff; color: #1890ff; font-size: 12px; padding: 2px 8px; border-radius: 4px; margin-bottom: 6px; font-weight: 500; }
.example-desc { font-size: 14px; color: var(--text-primary); margin-bottom: 8px; display: block; font-weight: 500; transition: color 0.3s; }

/* 笔记区 */
.note-textarea { width: 100%; min-height: 100px; background: var(--border-color); border: 1px solid var(--border-color); border-radius: 8px; padding: 10px; font-size: 14px; color: var(--text-primary); box-sizing: border-box; transition: all 0.3s; }
.note-textarea:focus { border-color: #007AFF; outline: none; background: var(--card-bg); }
.note-tip { font-size: 12px; color: var(--text-tertiary); margin-top: 5px; display: block; text-align: right; transition: color 0.3s; }
</style>