<template>
  <view class="container">
    <!-- 状态 1: 答题进行中 -->
    <view v-if="quizState === 'answering'" class="quiz-content">
      <!-- 顶部进度 -->
      <view class="progress-bar">
        <view class="progress-fill" :style="{ width: ((currentIndex + 1) / totalQuestions) * 100 + '%' }"></view>
      </view>
      <view class="progress-text">第 {{ currentIndex + 1 }} / {{ totalQuestions }} 题</view>

      <!-- 题目区域 -->
      <view class="question-card">
        <view class="question-type-tag">{{ currentQuestion.type === 'desc_to_cmd' ? '看描述猜命令' : '看命令猜作用' }}</view>
        <text class="question-text">{{ currentQuestion.question }}</text>
      </view>

      <!-- 选项区域 -->
      <view class="options-list">
        <view 
          v-for="(option, index) in currentQuestion.options" 
          :key="index"
          class="option-item"
          :class="getOptionClass(option)"
          @click="handleSelect(option)"
        >
          <text class="option-text">{{ option.text }}</text>
          <text v-if="option.isCorrect && hasAnswered" class="option-icon">✅</text>
          <text v-if="!option.isCorrect && option.isSelected && hasAnswered" class="option-icon">❌</text>
        </view>
      </view>

      <!-- 解析面板 (答完后显示) -->
      <view v-if="hasAnswered" class="analysis-panel">
        <view class="analysis-title">💡 知识点解析</view>
        <view class="analysis-syntax">
          <text class="label">标准语法：</text>
          <text class="code">{{ currentQuestion.correctCmd.syntax }}</text>
        </view>
        <view class="analysis-example" v-if="currentQuestion.correctCmd.examples && currentQuestion.correctCmd.examples[0]">
          <text class="label">典型示例：</text>
          <text class="code">{{ currentQuestion.correctCmd.examples[0].code }}</text>
        </view>
        <button class="next-btn" @click="goToNextQuestion">
          {{ currentIndex === totalQuestions - 1 ? '查看成绩' : '下一题' }}
        </button>
      </view>
    </view>

    <!-- 状态 2: 结算页面 -->
    <view v-else-if="quizState === 'finished'" class="result-content">
      <view class="result-card">
        <text class="result-emoji">{{ score === totalQuestions ? '' : score >= 2 ? '' : '💪' }}</text>
        <text class="result-title">今日挑战完成！</text>
        <view class="score-display">
          <text class="score-num">{{ score }}</text>
          <text class="score-total">/ {{ totalQuestions }}</text>
        </view>
        <text class="result-desc">
          {{ score === totalQuestions ? '全对！你已经是 Linux 达人了！' : '继续加油，熟能生巧！' }}
        </text>
      </view>

      <view class="action-buttons">
        <button class="action-btn primary" @click="reviewWrongAnswers" v-if="wrongAnswersInQuiz.length > 0">
          复习错题 ({{ wrongAnswersInQuiz.length }})
        </button>
        <button class="action-btn secondary" @click="resetQuiz">
          明日再来
        </button>
      </view>
    </view>

    <!-- 状态 3: 今日已完成提示 -->
    <view v-else-if="quizState === 'completed_today'" class="completed-content">
      <text class="big-emoji">🎉</text>
      <text class="title">今日挑战已完成</text>
      <text class="desc">你今天的得分是：{{ todayScore }} / {{ totalQuestions }}</text>
      <button class="action-btn primary" @click="reviewTodayQuiz" style="margin-top: 30px;">
        回顾今日题目
      </button>
    </view>

    <!-- 状态 4: 数据加载中 -->
    <view v-else class="loading-container">
      <text>正在加载题库...</text>
    </view>
  </view>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

const quizState = ref('loading') // 'loading', 'answering', 'finished', 'completed_today'
const totalQuestions = 3
const currentIndex = ref(0)
const hasAnswered = ref(false)
const score = ref(0)
const todayScore = ref(0)
const wrongAnswersInQuiz = ref([]) // 记录本次答错的命令ID

const questions = ref([])
const currentQuestion = computed(() => questions.value[currentIndex.value] || {})

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
    console.log('题库加载成功，共', commandsData.value.length, '道题')
  } catch (error) {
    console.error('加载题库失败:', error)
    uni.showToast({ title: '题库加载失败', icon: 'none' })
  }
}

// --- 核心算法 1: 基于日期的确定性随机数生成器 ---
const getDailySeed = () => {
  const today = new Date().toISOString().split('T')[0] // "2026-06-05"
  let hash = 0
  for (let i = 0; i < today.length; i++) {
    hash = ((hash << 5) - hash) + today.charCodeAt(i)
    hash = hash & hash
  }
  return Math.abs(hash)
}

// 简单的伪随机数生成器 (LCG)
const createRandom = (seed) => {
  let s = seed
  return () => {
    s = (s * 9301 + 49297) % 233280
    return s / 233280
  }
}

// --- 核心算法 2: Fisher-Yates 洗牌算法 ---
const shuffleArray = (array, randomFn) => {
  const arr = [...array]
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(randomFn() * (i + 1))
    ;[arr[i], arr[j]] = [arr[j], arr[i]]
  }
  return arr
}

// --- 业务逻辑: 生成每日题目 ---
const generateDailyQuestions = () => {
  if (commandsData.value.length < 4) return [] // 数据不足无法生成

  const seed = getDailySeed()
  const random = createRandom(seed)
  
  // 1. 随机抽取 3 个不重复的命令作为正确答案
  const shuffledCommands = shuffleArray(commandsData.value, random)
  const selectedCommands = shuffledCommands.slice(0, totalQuestions)
  
  // 2. 为每个正确答案生成题目和选项
  return selectedCommands.map(cmd => {
    // 随机决定题型 (50% 看描述猜命令, 50% 看命令猜作用)
    const type = random() > 0.5 ? 'desc_to_cmd' : 'cmd_to_desc'
    const questionText = type === 'desc_to_cmd' ? cmd.shortDesc : cmd.name
    
    // 生成 3 个干扰项
    const distractors = shuffledCommands
      .filter(c => c.id !== cmd.id)
      .slice(0, 3)
      .map(c => ({
        id: c.id,
        text: type === 'desc_to_cmd' ? c.name : c.shortDesc,
        isCorrect: false,
        isSelected: false
      }))
      
    // 正确选项
    const correctOption = {
      id: cmd.id,
      text: type === 'desc_to_cmd' ? cmd.name : cmd.shortDesc,
      isCorrect: true,
      isSelected: false
    }
    
    // 3. 洗牌 4 个选项
    const options = shuffleArray([correctOption, ...distractors], random)
    
    return {
      type,
      question: questionText,
      correctCmd: cmd,
      options
    }
  })
}

// --- 页面初始化 ---
onMounted(async () => {
  // 1. 先加载题库数据
  await loadData()
  
  // 2. 数据加载完成后再进行逻辑判断
  const storageKey = 'linux_daily_quiz'
  const quizData = JSON.parse(uni.getStorageSync(storageKey) || '{}')
  const today = new Date().toISOString().split('T')[0]
  
  if (quizData.date === today) {
    // 今天已经做过了
    todayScore.value = quizData.score || 0
    quizState.value = 'completed_today'
  } else {
    // 生成新题目
    questions.value = generateDailyQuestions()
    if (questions.value.length > 0) {
      quizState.value = 'answering'
    } else {
      uni.showToast({ title: '题库数据不足', icon: 'none' })
    }
  }
})

// --- 交互逻辑: 选择选项 ---
const handleSelect = (option) => {
  if (hasAnswered.value) return // 防止重复点击
  
  hasAnswered.value = true
  option.isSelected = true
  
  // 触发震动反馈
  uni.vibrateShort({ type: option.isCorrect ? 'light' : 'heavy' })
  
  if (option.isCorrect) {
    score.value++
    uni.showToast({ title: '回答正确！', icon: 'success', duration: 1000 })
  } else {
    // 记录错题
    const wrongId = currentQuestion.value.correctCmd.id
    wrongAnswersInQuiz.value.push(wrongId)
    
    // 优化：答错时立即同步到本地存储，防止中途退出丢失
    const wrongBook = JSON.parse(uni.getStorageSync('linux_wrong_answers') || '[]')
    if (!wrongBook.includes(wrongId)) {
      wrongBook.push(wrongId)
      uni.setStorageSync('linux_wrong_answers', JSON.stringify(wrongBook))
    }
    
    uni.showToast({ title: '回答错误', icon: 'none', duration: 1000 })
  }
}

// 获取选项的 CSS 类名
const getOptionClass = (option) => {
  if (!hasAnswered.value) return ''
  if (option.isCorrect) return 'correct'
  if (option.isSelected && !option.isCorrect) return 'wrong'
  return 'disabled'
}

// --- 交互逻辑: 下一题 / 结算 ---
const goToNextQuestion = () => {
  if (currentIndex.value < totalQuestions - 1) {
    currentIndex.value++
    hasAnswered.value = false
  } else {
    finishQuiz()
  }
}

const finishQuiz = () => {
  // 保存今日成绩
  const today = new Date().toISOString().split('T')[0]
  uni.setStorageSync('linux_daily_quiz', JSON.stringify({
    date: today,
    score: score.value,
    questions: questions.value // 保存题目以便回顾
  }))
  
  quizState.value = 'finished'
}

// --- 辅助功能: 重置与回顾 ---
const resetQuiz = () => {
  uni.switchTab({ url: '/pages/index/index' })
}

const reviewWrongAnswers = () => {
  uni.showToast({ title: '即将跳转到错题本...', icon: 'none' })
}

const reviewTodayQuiz = () => {
  const quizData = JSON.parse(uni.getStorageSync('linux_daily_quiz') || '{}')
  if (quizData.questions) {
    questions.value = quizData.questions
    quizState.value = 'answering'
    currentIndex.value = 0
    hasAnswered.value = false
    score.value = 0 // 回顾模式不计分
  }
}
</script>

<style scoped>
.container { padding: 20px; background-color: var(--bg-color); min-height: 100vh; display: flex; flex-direction: column; transition: background-color 0.3s; }
.loading-container { display: flex; justify-content: center; align-items: center; height: 100vh; color: var(--text-secondary); font-size: 16px; }

/* 进度条 */
.progress-bar { height: 6px; background: var(--border-color); border-radius: 3px; overflow: hidden; margin-bottom: 10px; transition: background-color 0.3s; }
.progress-fill { height: 100%; background: linear-gradient(90deg, #007AFF, #00C6FF); transition: width 0.3s ease; }
.progress-text { text-align: right; font-size: 12px; color: var(--text-tertiary); margin-bottom: 20px; transition: color 0.3s; }

/* 题目卡片 */
.question-card { background: var(--card-bg); padding: 25px 20px; border-radius: 16px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); margin-bottom: 25px; text-align: center; transition: background-color 0.3s; }
.question-type-tag { display: inline-block; background: #e6f7ff; color: #1890ff; font-size: 12px; padding: 4px 12px; border-radius: 12px; margin-bottom: 15px; font-weight: 500; }
.question-text { font-size: 18px; font-weight: bold; color: var(--text-primary); line-height: 1.5; display: block; transition: color 0.3s; }

/* 选项列表 */
.options-list { display: flex; flex-direction: column; gap: 12px; }
.option-item { background: var(--card-bg); padding: 16px 20px; border-radius: 12px; border: 2px solid var(--border-color); display: flex; justify-content: space-between; align-items: center; transition: all 0.2s; }
.option-item:active { transform: scale(0.98); }
.option-text { font-size: 15px; color: var(--text-primary); flex: 1; transition: color 0.3s; }
.option-icon { font-size: 18px; margin-left: 10px; }

/* 选项状态 */
.option-item.correct { background: #f6ffed; border-color: #52c41a; }
.option-item.correct .option-text { color: #389e0d; font-weight: bold; }
.option-item.wrong { background: #fff2f0; border-color: #ff4d4f; }
.option-item.wrong .option-text { color: #cf1322; }
.option-item.disabled { opacity: 0.6; }

/* 解析面板 */
.analysis-panel { margin-top: 25px; background: var(--card-bg); padding: 20px; border-radius: 16px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); animation: slideUp 0.3s ease; transition: background-color 0.3s; }
.analysis-title { font-size: 16px; font-weight: bold; color: var(--text-primary); margin-bottom: 15px; border-left: 4px solid #007AFF; padding-left: 10px; transition: color 0.3s; }
.analysis-syntax, .analysis-example { margin-bottom: 12px; }
.label { font-size: 13px; color: var(--text-secondary); display: block; margin-bottom: 4px; transition: color 0.3s; }
.code { font-family: monospace; background: var(--border-color); padding: 8px 12px; border-radius: 6px; font-size: 14px; color: #ff7b72; display: block; word-break: break-all; transition: all 0.3s; }
.next-btn { margin-top: 15px; background: #007AFF; color: white; border: none; border-radius: 25px; font-size: 16px; font-weight: bold; padding: 12px; width: 100%; }

/* 结算页 */
.result-content, .completed-content { flex: 1; display: flex; flex-direction: column; align-items: center; justify-content: center; }
.result-card, .completed-content { text-align: center; }
.result-emoji, .big-emoji { font-size: 60px; margin-bottom: 15px; }
.result-title, .title { font-size: 22px; font-weight: bold; color: var(--text-primary); margin-bottom: 10px; transition: color 0.3s; }
.score-display { margin: 20px 0; }
.score-num { font-size: 48px; font-weight: bold; color: #007AFF; }
.score-total { font-size: 20px; color: var(--text-tertiary); transition: color 0.3s; }
.result-desc, .desc { font-size: 15px; color: var(--text-secondary); transition: color 0.3s; }

.action-buttons { margin-top: 40px; width: 100%; display: flex; flex-direction: column; gap: 15px; }
.action-btn { width: 100%; padding: 14px; border-radius: 25px; font-size: 16px; font-weight: bold; border: none; }
.action-btn.primary { background: #007AFF; color: white; }
.action-btn.secondary { background: var(--border-color); color: var(--text-secondary); transition: all 0.3s; }

@keyframes slideUp {
  from { transform: translateY(20px); opacity: 0; }
  to { transform: translateY(0); opacity: 1; }
}
</style>