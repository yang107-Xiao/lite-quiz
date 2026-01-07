<script setup>
import { ref, computed, onMounted, nextTick, watch } from 'vue'
import localforage from 'localforage'
import * as XLSX from 'xlsx'
import confetti from 'canvas-confetti'
// import katex from 'katex' // 如果需要数学公式渲染，请取消注释并配置
// import 'katex/dist/katex.min.css'

/**
 * ==========================================
 * 数据结构与常量定义
 * ==========================================
 */
const STORAGE_KEY = 'lite-quiz-data-v2'

// 页面枚举
const PAGES = {
  DASHBOARD: 'dashboard',
  QUIZ: 'quiz',
  MISTAKES: 'mistakes'
}

// 鼓励文案
const ENCOURAGEMENTS = [
  "没关系，失败是成功之母！",
  "再试一次，你肯定行！",
  "这个问题确实有点难，记下来就好！",
  "别灰心，稳住心态！"
]

// 初始状态
const initialState = {
  questions: [],
  mistakeSet: [], // 存储题目ID
  stats: {
    totalAnswered: 0,
    correctCount: 0
  }
}

/**
 * ==========================================
 * 响应式状态
 * ==========================================
 */
const currentPage = ref(PAGES.DASHBOARD)
const appData = ref(JSON.parse(JSON.stringify(initialState)))
const currentQuizQueue = ref([]) // 当前刷题队列
const currentQuestionIndex = ref(0)
const selectedOption = ref(null) // 用户当前选择
const textAnswer = ref('') // 填空题答案
const isAnswered = ref(false) // 是否已提交
const isCorrect = ref(false) // 是否答对
const wrongStreak = ref(0) // 连续错误计数
const showEncouragement = ref('') // 显示鼓励语

// 错题复习相关
const isReviewMode = ref(false)

/**
 * ==========================================
 * 计算属性
 * ==========================================
 */
const totalQuestions = computed(() => appData.value.questions.length)
const mistakeCount = computed(() => appData.value.mistakeSet.length)
const completionRate = computed(() => {
  if (totalQuestions.value === 0) return 0
  // 这里简单用已答题数/总题数估算，或者根据 mastery_level > 0 计算
  const mastered = appData.value.questions.filter(q => q.mastery_level > 0).length
  return Math.round((mastered / totalQuestions.value) * 100)
})

const currentQuestion = computed(() => {
  if (!currentQuizQueue.value.length) return null
  const qId = currentQuizQueue.value[currentQuestionIndex.value]
  return appData.value.questions.find(q => q.id === qId)
})

/**
 * ==========================================
 * 持久化逻辑
 * ==========================================
 */
const saveData = () => {
  localforage.setItem(STORAGE_KEY, JSON.parse(JSON.stringify(appData.value)))
}

onMounted(async () => {
  const saved = await localforage.getItem(STORAGE_KEY)
  if (saved) {
    appData.value = saved
  }
})

/**
 * ==========================================
 * 业务逻辑：导入与管理
 * ==========================================
 */
const handleFileUpload = (event) => {
  const file = event.target.files[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = (e) => {
    try {
      const data = new Uint8Array(e.target.result)
      const workbook = XLSX.read(data, { type: 'array' })
      const firstSheetName = workbook.SheetNames[0]
      const worksheet = workbook.Sheets[firstSheetName]
      const jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 })

      // 解析 Excel (假设第一行是表头)
      // 格式: Type | Question | OptionA | ... | Answer | Analysis
      const newQuestions = []
      
      // 跳过表头，从第1行开始
      for (let i = 1; i < jsonData.length; i++) {
        const row = jsonData[i]
        if (!row || row.length < 2) continue

        // 简单的列映射 logic
        const type = row[0] ? row[0].toLowerCase() : 'single'
        const questionText = row[1]
        
        // 提取选项 (假设 C, D, E, F 列是选项)
        const options = []
        // 动态检测选项列，这里简化处理，假设最多4个选项
        if (row[2]) options.push(row[2]) // Option A
        if (row[3]) options.push(row[3]) // Option B
        if (row[4]) options.push(row[4]) // Option C
        if (row[5]) options.push(row[5]) // Option D

        // 答案在 G 列 (索引 6) 或 动态判断，这里假设紧跟选项后
        // 简单起见，假设 Excel 模板固定：Type, Question, OptA, OptB, OptC, OptD, Answer, Analysis
        const answer = row[6] ? String(row[6]) : ''
        const analysis = row[7] || '暂无解析'

        newQuestions.push({
          id: Date.now() + '_' + i,
          type: type.includes('填空') ? 'blank' : (type.includes('多选') ? 'multiple' : 'single'),
          question: questionText,
          options: options,
          answer: answer,
          analysis: analysis,
          mastery_level: 0
        })
      }

      appData.value.questions = newQuestions
      saveData()
      alert(`成功导入 ${newQuestions.length} 道题目！`)
    } catch (error) {
      console.error(error)
      alert('文件解析失败，请检查格式')
    }
  }
  reader.readAsArrayBuffer(file)
}

const exportData = () => {
  const dataStr = JSON.stringify(appData.value)
  const blob = new Blob([dataStr], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `lite-quiz-backup-${new Date().toISOString().slice(0,10)}.json`
  a.click()
}

/**
 * ==========================================
 * 业务逻辑：刷题
 * ==========================================
 */
const startQuiz = (mode = 'normal') => {
  isReviewMode.value = (mode === 'mistake')
  
  let queue = []
  if (mode === 'mistake') {
    queue = [...appData.value.mistakeSet]
  } else {
    // 随机抽取 20 题，或者全部
    const allIds = appData.value.questions.map(q => q.id)
    // Fisher-Yates Shuffle
    for (let i = allIds.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [allIds[i], allIds[j]] = [allIds[j], allIds[i]];
    }
    queue = allIds.slice(0, 20)
  }

  if (queue.length === 0) {
    alert(mode === 'mistake' ? '恭喜！目前没有错题需要复习。' : '还没有导入题目哦！')
    return
  }

  currentQuizQueue.value = queue
  currentQuestionIndex.value = 0
  resetQuestionState()
  currentPage.value = PAGES.QUIZ
}

const resetQuestionState = () => {
  selectedOption.value = null
  textAnswer.value = ''
  isAnswered.value = false
  isCorrect.value = false
  showEncouragement.value = ''
}

const submitAnswer = () => {
  if (isAnswered.value) return // 防止重复提交

  const q = currentQuestion.value
  let userCorrect = false

  if (q.type === 'blank') {
    const cleanUser = textAnswer.value.trim().toLowerCase()
    const cleanAns = q.answer.trim().toLowerCase()
    userCorrect = (cleanUser === cleanAns)
  } else {
    // 单选/判断逻辑
    // 如果是多选，这里需要改造成数组比较
    userCorrect = (String(selectedOption.value).trim() === String(q.answer).trim())
  }

  isCorrect.value = userCorrect
  isAnswered.value = true
  appData.value.stats.totalAnswered++

  if (userCorrect) {
    appData.value.stats.correctCount++
    wrongStreak.value = 0
    
    // 熟练度逻辑
    const qIndex = appData.value.questions.findIndex(item => item.id === q.id)
    if (qIndex !== -1) {
       // 答对 +1
       appData.value.questions[qIndex].mastery_level = Math.min(appData.value.questions[qIndex].mastery_level + 1, 3)
       
       // 如果熟练度 >= 3 且在错题集中，移除
       if (appData.value.questions[qIndex].mastery_level >= 3) {
         removeFromMistakes(q.id)
       }
    }

    // 动效
    confetti({
      particleCount: 100,
      spread: 70,
      origin: { y: 0.6 },
      colors: ['#6366F1', '#8B5CF6', '#10B981'] // 使用主题色
    })

    // 自动跳转 (可选)
    // setTimeout(nextQuestion, 1500)

  } else {
    wrongStreak.value++
    // 错题处理
    const qIndex = appData.value.questions.findIndex(item => item.id === q.id)
    if (qIndex !== -1) {
       appData.value.questions[qIndex].mastery_level = -1
    }
    if (!appData.value.mistakeSet.includes(q.id)) {
      appData.value.mistakeSet.push(q.id)
    }

    // 安慰文案
    if (wrongStreak.value >= 3) {
      showEncouragement.value = ENCOURAGEMENTS[Math.floor(Math.random() * ENCOURAGEMENTS.length)]
    }
  }

  saveData()
}

const nextQuestion = () => {
  if (currentQuestionIndex.value < currentQuizQueue.value.length - 1) {
    currentQuestionIndex.value++
    resetQuestionState()
  } else {
    alert('本组练习完成！')
    goHome()
  }
}

const goHome = () => {
  currentPage.value = PAGES.DASHBOARD
  saveData()
}

const removeFromMistakes = (id) => {
  appData.value.mistakeSet = appData.value.mistakeSet.filter(x => x !== id)
  // 如果在复习模式下，也要从当前队列移除（为了简单，暂不动态修改队列，只修数据）
  saveData()
}

</script>

<template>
  <div class="min-h-screen bg-gradient-to-br from-indigo-50 via-white to-purple-50 font-sans text-slate-900 p-4 md:p-6 transition-colors duration-500">
    
    <div v-if="currentPage === PAGES.DASHBOARD" class="max-w-md mx-auto space-y-8 pt-10">
      
      <div class="text-center space-y-2">
        <h1 class="text-4xl font-extrabold bg-clip-text text-transparent bg-gradient-to-r from-violet-600 to-indigo-600 tracking-tight">
          Lite Quiz
        </h1>
        <p class="text-slate-500 font-medium">极简刷题 · 快乐学习</p>
      </div>

      <div class="glass-card p-6 flex items-center justify-between">
        <div>
          <p class="text-sm text-slate-400 font-bold uppercase tracking-wider">题库进度</p>
          <p class="text-3xl font-extrabold text-slate-800 mt-1">
            {{ completionRate }}<span class="text-sm text-slate-400 ml-1">%</span>
          </p>
          <p class="text-xs text-slate-400 mt-1">总题数: {{ totalQuestions }}</p>
        </div>
        <div class="relative w-20 h-20">
          <svg class="w-full h-full transform -rotate-90">
            <circle cx="40" cy="40" r="36" stroke="currentColor" stroke-width="8" fill="transparent" class="text-indigo-100" />
            <circle cx="40" cy="40" r="36" stroke="currentColor" stroke-width="8" fill="transparent" 
              :stroke-dasharray="226" 
              :stroke-dashoffset="226 - (226 * completionRate) / 100" 
              class="text-violet-500 transition-all duration-1000 ease-out" 
              stroke-linecap="round" />
          </svg>
        </div>
      </div>

      <div v-if="mistakeCount > 0" 
           @click="startQuiz('mistake')"
           class="glass-card p-4 flex items-center justify-between cursor-pointer group hover:border-rose-200 transition-all">
        <div class="flex items-center gap-3">
          <div class="w-10 h-10 rounded-full bg-rose-100 flex items-center justify-center text-rose-600 font-bold shadow-sm">
            {{ mistakeCount }}
          </div>
          <div>
            <h3 class="font-bold text-slate-800 group-hover:text-rose-600 transition-colors">错题待复习</h3>
            <p class="text-xs text-slate-400">点击进入消灭错题模式</p>
          </div>
        </div>
        <div class="w-8 h-8 rounded-full bg-rose-50 flex items-center justify-center text-rose-400 group-hover:bg-rose-500 group-hover:text-white transition-all">
          ➜
        </div>
      </div>

      <div class="space-y-4 pt-4">
        <button @click="startQuiz('normal')" 
          class="btn-primary w-full py-4 text-lg shadow-indigo-500/30">
          开始今日刷题
        </button>
        
        <div class="grid grid-cols-2 gap-4">
          <label class="btn-secondary text-center cursor-pointer">
            导入题库
            <input type="file" class="hidden" accept=".xlsx" @change="handleFileUpload" />
          </label>
          <button @click="exportData" class="btn-secondary">
            备份进度
          </button>
        </div>
      </div>
    </div>

    <div v-else-if="currentPage === PAGES.QUIZ" class="max-w-lg mx-auto h-full flex flex-col pt-4">
      
      <div class="flex justify-between items-center mb-6 px-2">
        <div class="flex items-center gap-2">
          <span class="text-xs font-bold text-indigo-400 bg-indigo-50 px-2 py-1 rounded-lg uppercase tracking-wider">
             {{ isReviewMode ? 'Mistake Mode' : 'Quiz Mode' }}
          </span>
        </div>
        <div class="text-slate-400 font-bold text-sm">
          {{ currentQuestionIndex + 1 }} <span class="text-slate-200">/</span> {{ currentQuizQueue.length }}
        </div>
        <button @click="goHome" class="w-8 h-8 rounded-full bg-white text-slate-400 hover:text-rose-500 hover:bg-rose-50 flex items-center justify-center transition-all shadow-sm">
          ✕
        </button>
      </div>

      <div class="glass-card flex-1 flex flex-col p-6 mb-4 relative overflow-hidden">
        
        <div class="mb-8">
           <span class="text-xs font-bold text-slate-400 mb-2 block uppercase">
             {{ currentQuestion.type === 'blank' ? '填空题' : '单选题' }}
           </span>
           <h2 class="text-xl md:text-2xl font-extrabold text-slate-800 leading-snug">
             {{ currentQuestion.question }}
           </h2>
        </div>

        <div class="space-y-3 flex-1 overflow-y-auto custom-scrollbar">
          
          <template v-if="currentQuestion.type !== 'blank'">
            <div v-for="(opt, idx) in currentQuestion.options" :key="idx"
                 @click="!isAnswered && (selectedOption = opt)"
                 :class="[
                   'relative p-4 rounded-xl border-2 transition-all cursor-pointer group',
                   // Default State
                   !isAnswered && selectedOption !== opt ? 'border-transparent bg-slate-50 hover:bg-indigo-50/50 hover:border-indigo-100' : '',
                   // Selected State (Before Submit)
                   !isAnswered && selectedOption === opt ? 'border-indigo-500 bg-indigo-50 shadow-md shadow-indigo-200' : '',
                   // Correct State (After Submit)
                   isAnswered && opt === currentQuestion.answer ? 'border-emerald-500 bg-emerald-50 text-emerald-800 shadow-md' : '',
                   // Wrong State (After Submit)
                   isAnswered && selectedOption === opt && opt !== currentQuestion.answer ? 'border-rose-500 bg-rose-50 text-rose-800' : '',
                   // Dim others
                   isAnswered && opt !== currentQuestion.answer && selectedOption !== opt ? 'opacity-40 grayscale' : ''
                 ]">
              <div class="flex items-center gap-3">
                <div :class="[
                  'w-6 h-6 rounded-full border-2 flex items-center justify-center text-xs font-bold transition-colors',
                  !isAnswered && selectedOption === opt ? 'border-indigo-500 text-indigo-500' : 'border-slate-300 text-slate-400',
                  isAnswered && opt === currentQuestion.answer ? '!border-emerald-500 !bg-emerald-500 text-white' : '',
                  isAnswered && selectedOption === opt && opt !== currentQuestion.answer ? '!border-rose-500 !text-rose-500' : ''
                ]">
                  {{ ['A','B','C','D'][idx] || idx + 1 }}
                </div>
                <span class="font-medium">{{ opt }}</span>
              </div>
            </div>
          </template>

          <template v-else>
            <input 
              v-model="textAnswer"
              :disabled="isAnswered"
              type="text" 
              placeholder="请输入答案..."
              :class="[
                'w-full p-4 rounded-xl border-2 outline-none transition-all font-medium text-lg',
                !isAnswered ? 'border-slate-200 focus:border-indigo-500 focus:ring-4 focus:ring-indigo-500/10' : '',
                isAnswered && isCorrect ? 'border-emerald-500 bg-emerald-50 text-emerald-700' : '',
                isAnswered && !isCorrect ? 'border-rose-500 bg-rose-50 text-rose-700' : ''
              ]"
            />
          </template>
        </div>

        <transition name="fade-slide">
          <div v-if="isAnswered" 
            :class="[
              'mt-6 p-5 rounded-xl border border-opacity-50 relative overflow-hidden',
              isCorrect ? 'bg-emerald-50/50 border-emerald-100' : 'bg-rose-50/50 border-rose-100'
            ]">
            <div class="flex items-center gap-2 mb-2">
              <span class="text-2xl">{{ isCorrect ? '🎉' : '🥀' }}</span>
              <h4 :class="['font-bold text-lg', isCorrect ? 'text-emerald-700' : 'text-rose-700']">
                {{ isCorrect ? 'Bingo! 答对了' : 'Oops! 答错了' }}
              </h4>
            </div>
            
            <p v-if="!isCorrect" class="text-sm font-bold text-slate-500 mb-1">正确答案：</p>
            <p v-if="!isCorrect" class="text-lg font-mono font-bold text-slate-800 mb-3">{{ currentQuestion.answer }}</p>
            
            <p class="text-sm font-bold text-slate-500 mb-1">解析：</p>
            <p class="text-sm text-slate-600 leading-relaxed">{{ currentQuestion.analysis }}</p>

            <div v-if="showEncouragement" class="mt-3 py-2 px-3 bg-white/60 rounded-lg text-xs font-bold text-rose-500 text-center animate-bounce">
              {{ showEncouragement }}
            </div>

            <button v-if="isReviewMode" @click="removeFromMistakes(currentQuestion.id)" class="absolute top-2 right-2 text-slate-300 hover:text-rose-500 transition-colors p-2">
              <span class="sr-only">移除错题</span>
              🗑️
            </button>
          </div>
        </transition>

      </div>

      <div class="mt-4 pb-6">
        <button 
          v-if="!isAnswered" 
          @click="submitAnswer"
          :disabled="(currentQuestion.type === 'blank' && !textAnswer) || (currentQuestion.type !== 'blank' && !selectedOption)"
          class="btn-primary w-full py-4 text-lg shadow-indigo-500/30 disabled:opacity-50 disabled:cursor-not-allowed">
          确认提交
        </button>
        <button 
          v-else 
          @click="nextQuestion"
          :class="[
            'w-full py-4 text-lg rounded-full font-bold shadow-lg transition-transform hover:scale-105 active:scale-95 text-white',
            isCorrect ? 'bg-gradient-to-r from-emerald-500 to-teal-500 shadow-emerald-500/30' : 'bg-gradient-to-r from-indigo-500 to-violet-500 shadow-indigo-500/30'
          ]">
          {{ currentQuestionIndex < currentQuizQueue.length - 1 ? '下一题 ➜' : '查看结果' }}
        </button>
      </div>

    </div>
  </div>
</template>

<style>
/* Global Styles placed here for single-file convenience */
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;700;800&display=swap');

body {
  font-family: 'Inter', system-ui, -apple-system, sans-serif;
  @apply bg-slate-50; /* Fallback */
}

/* Custom Components using Tailwind Apply */
.glass-card {
  @apply bg-white/90 backdrop-blur-md border border-white/40 rounded-2xl shadow-xl shadow-indigo-500/10;
}

.btn-primary {
  @apply bg-gradient-to-r from-violet-600 to-indigo-600 text-white font-bold rounded-full shadow-lg transition-transform hover:scale-105 active:scale-95;
}

.btn-secondary {
  @apply bg-white text-slate-600 font-bold py-3 rounded-full border border-slate-200 shadow-sm transition-all hover:bg-slate-50 hover:text-indigo-600 active:scale-95 hover:border-indigo-100;
}

/* Transitions */
.fade-slide-enter-active,
.fade-slide-leave-active {
  transition: all 0.4s ease;
}
.fade-slide-enter-from {
  opacity: 0;
  transform: translateY(20px);
}
.fade-slide-leave-to {
  opacity: 0;
  transform: translateY(-20px);
}

.custom-scrollbar::-webkit-scrollbar {
  width: 6px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: transparent;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background-color: #e2e8f0;
  border-radius: 20px;
}
</style>