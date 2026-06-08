<script>
export default {
  onLaunch: function() {
    console.log('App Launch')
    
    // 1. 监听系统主题变化 (App 端)
    uni.onThemeChange((res) => {
      this.applyTheme(res.theme)
    })
    
    // 2. 初始化时获取当前系统主题
    const systemInfo = uni.getSystemInfoSync()
    if (systemInfo.theme) {
      this.applyTheme(systemInfo.theme)
    }
  },
  methods: {
    applyTheme(theme) {
      const isDark = theme === 'dark'
      
      // 动态设置原生导航栏颜色 (App 端)
      uni.setNavigationBarColor({
        frontColor: isDark ? '#ffffff' : '#000000',
        backgroundColor: isDark ? '#1A1A1A' : '#FFFFFF',
        animation: { duration: 300, timingFunc: 'easeIn' }
      })
      
      // 动态设置 TabBar 背景色和文字颜色 (App 端)
      uni.setTabBarStyle({
        backgroundColor: isDark ? '#1A1A1A' : '#FFFFFF',
        color: isDark ? '#999999' : '#999999',
        selectedColor: '#007AFF',
        borderStyle: isDark ? 'white' : 'black'
      })

      // 设置 H5 端的 body 背景色，防止白边
      if (typeof document !== 'undefined') {
        document.body.style.backgroundColor = isDark ? '#121212' : '#F5F7FA'
      }
    }
  }
}
</script>

<style>
/* ==========================================
   1. 全局 CSS 变量定义 (亮色模式默认值)
   ========================================== */
page {
  --bg-color: #F5F7FA;
  --card-bg: #FFFFFF;
  --text-primary: #333333;
  --text-secondary: #666666;
  --text-tertiary: #999999;
  --border-color: #F0F0F0;
  --theme-color: #007AFF;
  --code-bg: #282C34;
  --code-text: #ABB2BF;
}

/* ==========================================
   2. 暗黑模式变量覆盖
   ========================================== */
@media (prefers-color-scheme: dark) {
  page {
    --bg-color: #121212;
    --card-bg: #1E1E1E;
    --text-primary: #E0E0E0;
    --text-secondary: #A0A0A0;
    --text-tertiary: #666666;
    --border-color: #333333;
    --theme-color: #0A84FF;
    --code-bg: #1A1A1A;
    --code-text: #D4D4D4;
  }
}

/* ==========================================
   3. 全局基础样式与 TabBar 防遮挡修复
   ========================================== */
/* 统一页面容器基础样式，并强制预留底部 TabBar 空间 */
.container {
  background-color: var(--bg-color);
  color: var(--text-primary);
  min-height: 100vh;
  box-sizing: border-box;
  /* 核心修复：预留 65px 底部空间，确保内容滚动到底部时不会被原生 TabBar 遮挡 */
  padding-bottom: 65px !important; 
  transition: background-color 0.3s, color 0.3s;
}

/* 统一卡片样式类 (供各页面复用) */
.section-card {
  background-color: var(--card-bg);
  border-radius: 12px;
  padding: 15px;
  margin-bottom: 15px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.04);
  transition: background-color 0.3s;
}

/* 全局滚动条优化 (H5端) */
::-webkit-scrollbar {
  width: 0px;
  height: 0px;
}
</style>