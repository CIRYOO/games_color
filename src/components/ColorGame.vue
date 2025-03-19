<template>
  <div class="color-game">
    <a-card class="game-info" :bordered="false">
      <h2>颜色找不同</h2>
      <div class="settings">
        <a-form-item label="难度设置" class="difficulty-select">
          <a-select
            v-model:value="difficulty"
            style="width: 200px"
            @change="handleDifficultyChange"
            size="large"
          >
            <a-select-option value="easy">简单</a-select-option>
            <a-select-option value="medium">中等</a-select-option>
            <a-select-option value="hard">困难</a-select-option>
            <a-select-option value="expert">专家</a-select-option>
          </a-select>
        </a-form-item>
        <div class="difficulty-info">
          <a-tag :color="difficultyColors[difficulty]">当前难度: {{ difficultyText }}</a-tag>
          <a-tag color="#722ED1">分数倍率: {{ scoreMultiplier }}x</a-tag>
        </div>
      </div>
      <div class="stats">
        <div class="stat-item">
          <a-statistic 
            title="当前关卡" 
            :value="currentLevel"
            :value-style="{ color: '#1677ff' }"
          />
          <div class="sub-stat">
            剩余次数: {{ currentLevel < 6 ? maxPlayCount - playCount : '∞' }}
          </div>
        </div>
        <div class="stat-item">
          <a-statistic 
            title="得分" 
            :value="score"
            :value-style="{ color: '#722ED1' }"
          />
        </div>
        <div class="stat-item">
          <a-statistic 
            title="剩余时间" 
            :value="timeLeft" 
            suffix="秒"
            :value-style="{ color: timeLeft <= 10 ? '#ff4d4f' : '#52c41a' }"
          />
        </div>
      </div>
    </a-card>

    <div class="game-board" :style="gridStyle">
      <div
        v-for="(color, index) in colors"
        :key="index"
        class="color-block"
        :style="{ backgroundColor: color }"
        @click="checkColor(index)"
      ></div>
    </div>

    <a-modal
      v-model:open="gameOver"
      title="游戏结束"
      :closable="false"
      :maskClosable="false"
      :footer="null"
      centered
      wrapClassName="game-over-modal"
    >
      <a-result
        :title="`最终得分: ${score}`"
        :sub-title="currentLevel >= 6 ? '恭喜通关！' : '再接再厉！'"
      >
        <template #extra>
          <div class="history-scores">
            <h3>历史最高分</h3>
            <div class="history-list">
              <div v-for="(record, index) in historyScores" :key="index" class="history-item">
                <div class="medal">{{ ['🥇', '🥈', '🥉'][index] }}</div>
                <div class="score">{{ record.score }}分</div>
                <div class="detail">
                  <span class="difficulty">[{{ record.difficulty }}]</span>
                  <span class="date">{{ record.date }}</span>
                </div>
              </div>
              <div v-if="!historyScores.length" class="no-record">
                暂无记录
              </div>
            </div>
          </div>
          <a-space class="action-buttons">
            <!-- <a-button type="primary" @click="continueGame">继续游戏</a-button> -->
            <a-button @click="restartGame">重新开始</a-button>
          </a-space>
        </template>
      </a-result>
    </a-modal>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'

const currentLevel = ref(1)
const score = ref(0)
const timeLeft = ref(30)
const gameOver = ref(false)
const colors = ref([])
const timer = ref(null)
const differentColorIndex = ref(0)
const columnstest = ref(0)

// 难度设置
const difficulty = ref('medium')
const difficultySettings = {
  easy: { variation: 25, multiplier: 1, text: '简单' },
  medium: { variation: 15, multiplier: 2, text: '中等' },
  hard: { variation: 8, multiplier: 3, text: '困难' },
  expert: { variation: 4, multiplier: 5, text: '专家' }
}

// 计算属性
const difficultyText = computed(() => difficultySettings[difficulty.value].text)
const scoreMultiplier = computed(() => difficultySettings[difficulty.value].multiplier)

// 修改计算每关的方块数量
const blockCount = computed(() => {
  // 计算 (N+1)² 个方块，其中N是当前关卡数
  const n = currentLevel.value + 1
  return n * n
})

// 修改网格样式计算
const gridStyle = computed(() => {
  const columns = Math.sqrt(blockCount.value)
  const padding = 10
  // 计算容器最大可用宽度
  const maxContainerWidth = Math.min(window.innerWidth * 0.9 - (padding * 2), 400)
  // 计算间隙总宽度
  const gapSize = 4
  const totalGapWidth = gapSize * (columns - 1)
  // 计算单个方块的大小
  const blockSize = Math.floor((maxContainerWidth - totalGapWidth) / columns)
  // 计算实际容器大小
  const containerSize = blockSize * columns + totalGapWidth
  
  return {
    display: 'flex',
    flexWrap: 'wrap',
    gap: `${gapSize}px`,
    width: `${containerSize}px`,
    height: `${containerSize}px`,
    margin: '0 auto',
    backgroundColor: '#f0f0f0',
    borderRadius: '8px',
    padding: `${padding}px`
  }
})

// 生成柔和的颜色
const generateSoftColor = () => {
  // 柔和的颜色范围
  const softColors = [
    // 柔和的蓝色系
    { r: [100, 150], g: [130, 180], b: [160, 200] },
    // 柔和的绿色系
    { r: [120, 170], g: [150, 190], b: [120, 160] },
    // 柔和的棕色系
    { r: [170, 200], g: [140, 170], b: [110, 150] },
    // 柔和的紫色系
    { r: [150, 180], g: [130, 160], b: [170, 200] },
  ]
  
  const colorRange = softColors[Math.floor(Math.random() * softColors.length)]
  const r = Math.floor(Math.random() * (colorRange.r[1] - colorRange.r[0]) + colorRange.r[0])
  const g = Math.floor(Math.random() * (colorRange.g[1] - colorRange.g[0]) + colorRange.g[0])
  const b = Math.floor(Math.random() * (colorRange.b[1] - colorRange.b[0]) + colorRange.b[0])
  return `rgb(${r}, ${g}, ${b})`
}

// 修改随机颜色生成函数
const generateRandomColor = () => {
  // 前三关使用柔和的颜色
  if (currentLevel.value <= 3) {
    return generateSoftColor()
  }
  
  // 之后的关卡使用完全随机的颜色
  const r = Math.floor(Math.random() * 256)
  const g = Math.floor(Math.random() * 256)
  const b = Math.floor(Math.random() * 256)
  return `rgb(${r}, ${g}, ${b})`
}

// 生成相似颜色
const generateSimilarColor = (baseColor) => {
  const rgb = baseColor.match(/\d+/g).map(Number)
  const variation = difficultySettings[difficulty.value].variation
  
  // 前三关使用更小的颜色变化范围
  const actualVariation = currentLevel.value <= 3 
    ? Math.min(variation, 15) // 限制最大变化范围
    : variation
    
  const newRgb = rgb.map(value => {
    const newValue = value + (Math.random() * actualVariation * 2 - actualVariation)
    return Math.min(255, Math.max(0, Math.floor(newValue)))
  })
  return `rgb(${newRgb[0]}, ${newRgb[1]}, ${newRgb[2]})`
}

// 添加游玩次数和历史得分记录
const playCount = ref(0)
const maxPlayCount = computed(() => currentLevel.value < 6 ? currentLevel.value * 2 : Infinity)
const historyScores = ref(JSON.parse(localStorage.getItem('colorGameScores') || '[]'))

// 更新历史得分
const updateHistoryScores = (isLevelChange = false) => {
  if (score.value > 0 || isLevelChange) {
    historyScores.value.push({
      score: score.value,
      difficulty: difficultyText.value,
      date: new Date().toLocaleString(),
      reason: isLevelChange ? '难度切换' : '游戏结束'
    })
    // 按分数降序排序并只保留前3名
    historyScores.value.sort((a, b) => b.score - a.score)
    historyScores.value = historyScores.value.slice(0, 3)
    // 保存到本地存储
    localStorage.setItem('colorGameScores', JSON.stringify(historyScores.value))
  }
}

// 初始化关卡
const initializeLevel = () => {
  const baseColor = generateRandomColor()
  colors.value = new Array(blockCount.value).fill(baseColor)
  differentColorIndex.value = Math.floor(Math.random() * blockCount.value)
  colors.value[differentColorIndex.value] = generateSimilarColor(baseColor)
  timeLeft.value = 30
  startTimer()
}

// 开始计时器
const startTimer = () => {
  clearInterval(timer.value)
  timer.value = setInterval(() => {
    timeLeft.value--
    if (timeLeft.value <= 0) {
      endGame()
    }
  }, 1000)
}

// 检查选择的颜色
const checkColor = (index) => {
  if (gameOver.value) return

  playCount.value++

  if (index === differentColorIndex.value) {
    score.value += currentLevel.value * 10 * difficultySettings[difficulty.value].multiplier
    if (currentLevel.value < 6) {
      if (playCount.value >= maxPlayCount.value) {
        currentLevel.value++
        playCount.value = 0
      }
      initializeLevel()
    } else {
      // 最后一关可以无限游戏
      initializeLevel()
    }
  } else {
    endGame()
  }
}

// 结束游戏
const endGame = (won = false) => {
  clearInterval(timer.value)
  gameOver.value = true
  if (won) {
    score.value += 100 * difficultySettings[difficulty.value].multiplier
  }
  updateHistoryScores(false)
}

// 重新开始游戏
const restartGame = () => {
  currentLevel.value = 1
  score.value = 0
  playCount.value = 0
  gameOver.value = false
  initializeLevel()
}

// 继续游戏
const continueGame = () => {
  gameOver.value = false
  timeLeft.value = 30
  if (currentLevel.value < 6) {
    playCount.value = 0
  }
  initializeLevel()
}

// 处理难度变更
const handleDifficultyChange = () => {
  if (!gameOver.value) {
    updateHistoryScores()
    currentLevel.value = 1
    score.value = 0
    playCount.value = 0
    initializeLevel()
  }
}

// 添加难度颜色映射
const difficultyColors = {
  easy: '#52c41a',    // 绿色
  medium: '#1677ff',  // 蓝色
  hard: '#faad14',    // 橙色
  expert: '#ff4d4f'   // 红色
}

// 监听 blockCount 变化，更新 CSS 变量
watch(() => blockCount.value, (newValue) => {
  const columns = Math.sqrt(newValue)
  document.documentElement.style.setProperty('--columns', columns)
})

// 组件挂载时初始化游戏
onMounted(() => {
  const columns = Math.sqrt(blockCount.value)
  document.documentElement.style.setProperty('--columns', columns)
  initializeLevel()
  window.addEventListener('resize', initializeLevel)
})

// 组件卸载时清理定时器
onUnmounted(() => {
  clearInterval(timer.value)
  window.removeEventListener('resize', initializeLevel)
})
</script>

<style scoped>
.color-game {
  max-width: 800px;
  margin: 0 auto;
  padding: 10px;
  box-sizing: border-box;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  gap: 20px;
  background-color: #48bcff;
}

.game-info {
  background: linear-gradient(145deg, #ffffff, #f0f2f5);
  border-radius: 16px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  margin-bottom: 0; /* 移除底部边距 */
}

.game-info h2 {
  font-size: 1.5rem; /* 减小标题大小 */
  text-align: center;
  margin-bottom: 16px;
  font-weight: 600;
  background: linear-gradient(45deg, #1677ff, #722ED1);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.settings {
  background-color: #fafafa;
  padding: 12px; /* 减小内边距 */
  border-radius: 12px;
  margin-bottom: 16px; /* 减小底部边距 */
}

.difficulty-select {
  margin-bottom: 12px;
}

.difficulty-select :deep(.ant-form-item-label) {
  font-weight: 500;
  color: #1677ff;
}

.difficulty-info {
  display: flex;
  justify-content: center;
  gap: 16px;
  margin-top: 16px;
}

.stats {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
  padding: 12px; /* 减小内边距 */
  background-color: #fafafa;
  border-radius: 12px;
}

.stat-item {
  padding: 12px; /* 减小内边距 */
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
}

.game-board {
  margin: auto;
  background-color: #f0f0f0;
  padding: 10px;
  border-radius: 8px;
  box-sizing: border-box;
  width: min(90vw, 400px);
  height: min(90vw, 400px);
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
}

.color-block {
  flex: 1 1 calc(100% / var(--columns) - 4px); /* 使用CSS变量控制列数 */
  max-width: calc(100% / var(--columns) - 4px);
  aspect-ratio: 1;
  border-radius: 8px;
  cursor: pointer;
  touch-action: manipulation;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  background-color: inherit;
}

/* 只在非触摸设备上启用悬停效果 */
@media (hover: hover) {
  .color-block:hover {
    transform: scale(1.005);
  }
}

/* 触摸设备上的点击效果 */
@media (hover: none) {
  .color-block:active {
    opacity: 0.8;
  }
}

/* 移动端适配 */
@media (max-width: 480px) {
  .game-info h2 {
    font-size: 1.25rem;
    margin-bottom: 12px;
  }

  .stats {
    grid-template-columns: repeat(3, 1fr); /* 在移动端保持三列布局 */
    gap: 8px;
  }

  .stat-item {
    padding: 8px;
  }
  
  .stat-item :deep(.ant-statistic-title) {
    font-size: 12px; /* 减小统计标题字体 */
    margin-bottom: 4px;
  }
  
  .stat-item :deep(.ant-statistic-content) {
    font-size: 16px; /* 减小统计数值字体 */
  }
  
  .settings {
    padding: 10px;
  }
  
  .history-list {
    gap: 8px;
  }

  .history-item {
    padding: 8px;
    min-width: 100px;
  }

  .medal {
    font-size: 20px;
  }

  .score {
    font-size: 1rem;
  }
}

/* 平板和桌面端的优化 */
@media (min-width: 768px) {
  .color-game {
    padding: 20px;
    gap: 30px;
  }
  

}

.sub-stat {
  font-size: 12px;
  color: #666;
  margin-top: 4px;
  text-align: center;
}

.history-scores {
  margin: 12px 0;
}

.history-scores h3 {
  margin-bottom: 12px;
}

.history-list {
  display: flex;
  justify-content: center;
  gap: 12px;
  flex-wrap: wrap;
}

.history-item {
  background: #f5f5f5;
  padding: 8px;
  border-radius: 8px;
  min-width: 100px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  transition: transform 0.2s ease;
}

.history-item:hover {
  transform: translateY(-2px);
}

.medal {
  font-size: 20px;
  margin-bottom: 2px;
}

.score {
  color: #1677ff;
  font-weight: bold;
  font-size: 1.2rem;
}

.detail {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2px;
  font-size: 12px;
}

.difficulty {
  color: #666;
}

.date {
  color: #999;
}

.no-record {
  color: #999;
  padding: 20px;
  text-align: center;
  width: 100%;
}

.action-buttons {
  margin-top: 16px;
}

/* 添加弹窗样式 */
:deep(.game-over-modal .ant-modal-content) {
  max-height: 80vh;
  overflow-y: auto;
}

:deep(.game-over-modal .ant-modal) {
  top: 50%;
  transform: translateY(-50%);
  padding-bottom: 0;
}

/* 移动端适配 */
@media (max-width: 480px) {
  :deep(.game-over-modal .ant-modal-content) {
    max-height: 85vh; /* 移动端稍微增加高度 */
  }
  
  .history-scores {
    margin: 8px 0;
  }
  
  .history-item {
    padding: 6px;
  }
}
</style> 