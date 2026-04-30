<template>
  <div class="game-container">
    <div class="background-animate">
      <div v-for="i in 20" :key="i" class="cloud" :style="getCloudStyle(i)"></div>
    </div>
    
    <canvas ref="gameCanvas" class="game-canvas"></canvas>
    
    <div class="ui-overlay">
      <div class="score-panel glass-panel">
        <div class="score-item">
          <span class="score-label">层数</span>
          <span class="score-value">{{ currentFloor }}</span>
        </div>
        <div class="score-divider"></div>
        <div class="score-item">
          <span class="score-label">得分</span>
          <span class="score-value">{{ score }}</span>
        </div>
        <div class="score-divider"></div>
        <div class="score-item">
          <span class="score-label">最高</span>
          <span class="score-value highlight">{{ highScore }}</span>
        </div>
      </div>
      
      <div class="control-buttons" v-show="gameState !== 'idle'">
        <button 
          v-if="gameState === 'playing'" 
          class="control-btn glass-btn" 
          @click="pauseGame"
        >
          <span class="btn-icon">⏸</span>
          <span class="btn-text">暂停</span>
        </button>
        <button 
          v-else-if="gameState === 'paused'" 
          class="control-btn glass-btn" 
          @click="resumeGame"
        >
          <span class="btn-icon">▶</span>
          <span class="btn-text">继续</span>
        </button>
        <button 
          class="control-btn glass-btn" 
          @click="restartGame"
        >
          <span class="btn-icon">🔄</span>
          <span class="btn-text">重来</span>
        </button>
      </div>
    </div>
    
    <div class="start-screen" v-show="gameState === 'idle'">
      <div class="start-panel glass-panel">
        <h1 class="game-title">是男人就下100层</h1>
        <p class="game-subtitle">挑战你的极限！</p>
        <div class="high-score-display" v-if="highScore > 0">
          <span class="high-score-label">历史最高分：</span>
          <span class="high-score-value">{{ highScore }}</span>
        </div>
        <button class="start-btn glass-btn" @click="startGame">
          <span class="btn-icon">🎮</span>
          <span class="btn-text">开始游戏</span>
        </button>
        <div class="controls-hint">
          <p>🎯 键盘：← → 方向键</p>
          <p>📱 触屏：左右滑动或点击屏幕</p>
        </div>
      </div>
    </div>
    
    <div class="pause-screen" v-show="gameState === 'paused'">
      <div class="pause-panel glass-panel">
        <h2 class="pause-title">游戏暂停</h2>
        <div class="pause-stats">
          <p>当前层数：<span class="highlight">{{ currentFloor }}</span></p>
          <p>当前得分：<span class="highlight">{{ score }}</span></p>
        </div>
        <div class="pause-buttons">
          <button class="control-btn glass-btn primary" @click="resumeGame">
            <span class="btn-icon">▶</span>
            <span class="btn-text">继续游戏</span>
          </button>
          <button class="control-btn glass-btn" @click="restartGame">
            <span class="btn-icon">🔄</span>
            <span class="btn-text">重新开始</span>
          </button>
        </div>
      </div>
    </div>
    
    <div class="gameover-screen" v-show="gameState === 'gameover'">
      <div class="gameover-panel glass-panel">
        <h2 class="gameover-title">游戏结束</h2>
        <div class="gameover-stats">
          <div class="stat-item">
            <span class="stat-label">到达层数</span>
            <span class="stat-value highlight">{{ currentFloor }}</span>
          </div>
          <div class="stat-item">
            <span class="stat-label">本局得分</span>
            <span class="stat-value highlight">{{ score }}</span>
          </div>
          <div class="stat-item" v-if="isNewHighScore">
            <span class="stat-label new-record">🎉 新纪录！</span>
          </div>
          <div class="stat-item">
            <span class="stat-label">历史最高</span>
            <span class="stat-value">{{ highScore }}</span>
          </div>
        </div>
        <button class="restart-btn glass-btn primary" @click="restartGame">
          <span class="btn-icon">🔄</span>
          <span class="btn-text">再来一局</span>
        </button>
      </div>
    </div>
    
    <div class="touch-areas" v-show="gameState === 'playing'">
      <div class="touch-area left" @touchstart.prevent="handleTouchStart('left')" @touchend.prevent="handleTouchEnd"></div>
      <div class="touch-area right" @touchstart.prevent="handleTouchStart('right')" @touchend.prevent="handleTouchEnd"></div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const gameCanvas = ref(null)
let canvas = null
let ctx = null

const gameState = ref('idle')
const score = ref(0)
const currentFloor = ref(0)
const highScore = ref(0)
const isNewHighScore = ref(false)

let animationId = null
let lastTime = 0
let cameraY = 0
let highestPlayerY = Infinity

const PLATFORM_TYPES = {
  NORMAL: 'normal',
  DISAPPEARING: 'disappearing',
  BOUNCY: 'bouncy',
  BONUS: 'bonus'
}

const COLORS = {
  player: {
    body: '#FFB6C1',
    bodyShadow: '#FF69B4',
    face: '#FFE4E1',
    eyes: '#333',
    blush: '#FFB6C1',
    outline: '#FF1493'
  },
  platforms: {
    normal: { top: '#A8E6CF', bottom: '#88D4AB', glow: '#7BC99F' },
    disappearing: { top: '#FFB3B3', bottom: '#FF8080', glow: '#FF6666' },
    bouncy: { top: '#B3E0FF', bottom: '#80C8FF', glow: '#66B2FF' },
    bonus: { top: '#FFE066', bottom: '#FFD700', glow: '#FFC700' }
  }
}

const DIFFICULTY_CONFIG = {
  baseScrollSpeed: 1.5,
  maxScrollSpeed: 5.0,
  scrollSpeedIncrement: 0.15,
  
  baseGravity: 0.4,
  maxGravity: 0.8,
  gravityIncrement: 0.03,
  
  basePlatformWidth: 140,
  minPlatformWidth: 60,
  platformWidthDecrement: 5,
  
  basePlatformSpacing: 90,
  maxPlatformSpacing: 150,
  platformSpacingIncrement: 5,
  
  baseMoveSpeed: 1.5,
  maxMoveSpeed: 4.0,
  moveSpeedIncrement: 0.2,
  
  movingPlatformChance: 0.25,
  maxMovingPlatformChance: 0.6,
  movingPlatformChanceIncrement: 0.03
}

let difficultyParams = {
  scrollSpeed: DIFFICULTY_CONFIG.baseScrollSpeed,
  gravity: DIFFICULTY_CONFIG.baseGravity,
  platformWidth: DIFFICULTY_CONFIG.basePlatformWidth,
  platformSpacing: DIFFICULTY_CONFIG.basePlatformSpacing,
  moveSpeed: DIFFICULTY_CONFIG.baseMoveSpeed,
  movingPlatformChance: DIFFICULTY_CONFIG.movingPlatformChance
}

const player = {
  x: 0,
  y: 0,
  width: 36,
  height: 48,
  velocityX: 0,
  velocityY: 0,
  maxSpeed: 7,
  acceleration: 0.7,
  friction: 0.92,
  facingRight: true,
  animFrame: 0,
  standingPlatform: null
}

let platforms = []
let particles = []

const keys = {
  left: false,
  right: false
}

const cloudPositions = Array.from({ length: 20 }, () => ({
  x: Math.random() * 100,
  y: Math.random() * 100,
  size: 20 + Math.random() * 60,
  delay: Math.random() * 20,
  duration: 15 + Math.random() * 20
}))

const getCloudStyle = (index) => {
  const cloud = cloudPositions[index - 1]
  return {
    left: `${cloud.x}%`,
    top: `${cloud.y}%`,
    width: `${cloud.size}px`,
    height: `${cloud.size * 0.5}px`,
    animationDelay: `${cloud.delay}s`,
    animationDuration: `${cloud.duration}s`
  }
}

const loadHighScore = () => {
  const saved = localStorage.getItem('down100floors_highscore')
  if (saved) {
    highScore.value = parseInt(saved, 10)
  }
}

const saveHighScore = () => {
  if (score.value > highScore.value) {
    highScore.value = score.value
    isNewHighScore.value = true
    localStorage.setItem('down100floors_highscore', highScore.value.toString())
  }
}

const resetDifficulty = () => {
  difficultyParams = {
    scrollSpeed: DIFFICULTY_CONFIG.baseScrollSpeed,
    gravity: DIFFICULTY_CONFIG.baseGravity,
    platformWidth: DIFFICULTY_CONFIG.basePlatformWidth,
    platformSpacing: DIFFICULTY_CONFIG.basePlatformSpacing,
    moveSpeed: DIFFICULTY_CONFIG.baseMoveSpeed,
    movingPlatformChance: DIFFICULTY_CONFIG.movingPlatformChance
  }
}

const updateDifficulty = () => {
  const level = Math.floor(currentFloor.value / 5)
  
  difficultyParams.scrollSpeed = Math.min(
    DIFFICULTY_CONFIG.maxScrollSpeed,
    DIFFICULTY_CONFIG.baseScrollSpeed + level * DIFFICULTY_CONFIG.scrollSpeedIncrement
  )
  
  difficultyParams.gravity = Math.min(
    DIFFICULTY_CONFIG.maxGravity,
    DIFFICULTY_CONFIG.baseGravity + level * DIFFICULTY_CONFIG.gravityIncrement
  )
  
  difficultyParams.platformWidth = Math.max(
    DIFFICULTY_CONFIG.minPlatformWidth,
    DIFFICULTY_CONFIG.basePlatformWidth - level * DIFFICULTY_CONFIG.platformWidthDecrement
  )
  
  difficultyParams.platformSpacing = Math.min(
    DIFFICULTY_CONFIG.maxPlatformSpacing,
    DIFFICULTY_CONFIG.basePlatformSpacing + level * DIFFICULTY_CONFIG.platformSpacingIncrement
  )
  
  difficultyParams.moveSpeed = Math.min(
    DIFFICULTY_CONFIG.maxMoveSpeed,
    DIFFICULTY_CONFIG.baseMoveSpeed + level * DIFFICULTY_CONFIG.moveSpeedIncrement
  )
  
  difficultyParams.movingPlatformChance = Math.min(
    DIFFICULTY_CONFIG.maxMovingPlatformChance,
    DIFFICULTY_CONFIG.movingPlatformChance + level * DIFFICULTY_CONFIG.movingPlatformChanceIncrement
  )
}

const initCanvas = () => {
  canvas = gameCanvas.value
  ctx = canvas.getContext('2d')
  resizeCanvas()
}

const resizeCanvas = () => {
  const container = canvas.parentElement
  canvas.width = container.clientWidth
  canvas.height = container.clientHeight
  
  if (gameState.value === 'idle') {
    player.x = canvas.width / 2 - player.width / 2
    player.y = canvas.height * 0.4
  }
}

const getRandomPlatformType = () => {
  const rand = Math.random()
  const baseChance = 0.6
  const specialChance = 0.4 / 4
  
  if (rand < baseChance) return PLATFORM_TYPES.NORMAL
  if (rand < baseChance + specialChance) return PLATFORM_TYPES.DISAPPEARING
  if (rand < baseChance + specialChance * 2) return PLATFORM_TYPES.BOUNCY
  if (rand < baseChance + specialChance * 3) return PLATFORM_TYPES.BONUS
  return PLATFORM_TYPES.NORMAL
}

const createPlatform = (y, isStart = false) => {
  const width = isStart ? 160 : difficultyParams.platformWidth + Math.random() * 30
  const type = isStart ? PLATFORM_TYPES.NORMAL : getRandomPlatformType()
  const isMoving = !isStart && Math.random() < difficultyParams.movingPlatformChance
  
  return {
    x: isStart 
      ? (canvas.width - width) / 2 
      : Math.random() * (canvas.width - width),
    y: y,
    width: width,
    height: 18,
    type: type,
    opacity: 1,
    isDisappearing: false,
    disappearTimer: 0,
    velocityX: isMoving 
      ? (Math.random() > 0.5 ? 1 : -1) * (difficultyParams.moveSpeed * (0.5 + Math.random() * 0.5))
      : 0,
    touched: false,
    floorCounted: false
  }
}

const initPlatforms = () => {
  platforms = []
  
  const startY = canvas.height * 0.6
  const startPlatform = createPlatform(startY, true)
  platforms.push(startPlatform)
  
  let y = startY - difficultyParams.platformSpacing
  
  while (y > -difficultyParams.platformSpacing * 8) {
    platforms.push(createPlatform(y))
    y -= difficultyParams.platformSpacing
  }
}

const initPlayer = () => {
  const startPlatform = platforms.find(p => p.touched === false && p.type === PLATFORM_TYPES.NORMAL) || platforms[0]
  
  player.x = startPlatform.x + (startPlatform.width - player.width) / 2
  player.y = startPlatform.y - player.height
  player.velocityX = 0
  player.velocityY = -10
  player.standingPlatform = startPlatform
  player.facingRight = true
}

const startGame = () => {
  gameState.value = 'playing'
  score.value = 0
  currentFloor.value = 0
  isNewHighScore.value = false
  particles = []
  cameraY = 0
  highestPlayerY = Infinity
  
  resetDifficulty()
  initPlatforms()
  initPlayer()
  
  lastTime = performance.now()
  gameLoop()
}

const pauseGame = () => {
  if (gameState.value === 'playing') {
    gameState.value = 'paused'
    if (animationId) {
      cancelAnimationFrame(animationId)
      animationId = null
    }
  }
}

const resumeGame = () => {
  if (gameState.value === 'paused') {
    gameState.value = 'playing'
    lastTime = performance.now()
    gameLoop()
  }
}

const restartGame = () => {
  if (animationId) {
    cancelAnimationFrame(animationId)
    animationId = null
  }
  startGame()
}

const gameOver = () => {
  gameState.value = 'gameover'
  if (animationId) {
    cancelAnimationFrame(animationId)
    animationId = null
  }
  saveHighScore()
}

const handleKeyDown = (e) => {
  if (e.key === 'ArrowLeft' || e.key === 'a' || e.key === 'A') {
    keys.left = true
  }
  if (e.key === 'ArrowRight' || e.key === 'd' || e.key === 'D') {
    keys.right = true
  }
  if (e.key === 'Escape') {
    if (gameState.value === 'playing') {
      pauseGame()
    } else if (gameState.value === 'paused') {
      resumeGame()
    }
  }
}

const handleKeyUp = (e) => {
  if (e.key === 'ArrowLeft' || e.key === 'a' || e.key === 'A') {
    keys.left = false
  }
  if (e.key === 'ArrowRight' || e.key === 'd' || e.key === 'D') {
    keys.right = false
  }
}

const handleTouchStart = (direction) => {
  if (direction === 'left') {
    keys.left = true
    keys.right = false
  } else {
    keys.right = true
    keys.left = false
  }
}

const handleTouchEnd = () => {
  keys.left = false
  keys.right = false
}

const createParticles = (x, y, color, count = 10) => {
  for (let i = 0; i < count; i++) {
    particles.push({
      x: x,
      y: y,
      velocityX: (Math.random() - 0.5) * 6,
      velocityY: (Math.random() - 0.5) * 6 - 2,
      size: 3 + Math.random() * 4,
      color: color,
      life: 1,
      decay: 0.025 + Math.random() * 0.015
    })
  }
}

const updateParticles = () => {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i]
    p.x += p.velocityX
    p.y += p.velocityY
    p.velocityY += 0.15
    p.life -= p.decay
    
    if (p.life <= 0) {
      particles.splice(i, 1)
    }
  }
}

const checkGameOver = () => {
  const screenY = player.y - cameraY
  
  if (screenY > canvas.height + 50) {
    return true
  }
  
  return false
}

const updatePlayer = (dt) => {
  if (keys.left) {
    player.velocityX -= player.acceleration
    player.facingRight = false
  }
  if (keys.right) {
    player.velocityX += player.acceleration
    player.facingRight = true
  }
  
  player.velocityX *= player.friction
  
  if (Math.abs(player.velocityX) > player.maxSpeed) {
    player.velocityX = player.maxSpeed * Math.sign(player.velocityX)
  }
  
  player.velocityY += difficultyParams.gravity
  
  player.x += player.velocityX
  
  if (player.x + player.width < 0) {
    player.x = canvas.width
  } else if (player.x > canvas.width) {
    player.x = -player.width
  }
  
  if (Math.abs(player.velocityX) > 0.3) {
    player.animFrame += 0.15
  }
}

const handlePlatformCollision = () => {
  player.standingPlatform = null
  
  const playerBottom = player.y + player.height
  const playerCenterX = player.x + player.width / 2
  const prevBottom = playerBottom - player.velocityY
  
  for (const platform of platforms) {
    if (platform.opacity < 0.3) continue
    
    if (player.velocityY > 0 &&
        prevBottom <= platform.y &&
        playerBottom >= platform.y &&
        playerCenterX >= platform.x - 5 &&
        playerCenterX <= platform.x + platform.width + 5) {
      
      player.y = platform.y - player.height
      player.standingPlatform = platform
      
      if (!platform.touched) {
        platform.touched = true
      }
      
      if (!platform.floorCounted) {
        platform.floorCounted = true
        currentFloor.value++
        updateDifficulty()
        
        const baseScore = 10
        const bonusScore = Math.floor(Math.random() * 20)
        score.value += baseScore + bonusScore
        createParticles(player.x + player.width / 2, player.y + player.height, '#FFD700', 5)
      }
      
      switch (platform.type) {
        case PLATFORM_TYPES.BOUNCY:
          player.velocityY = -16
          player.standingPlatform = null
          createParticles(player.x + player.width / 2, player.y + player.height, '#80C8FF', 15)
          break
          
        case PLATFORM_TYPES.DISAPPEARING:
          player.velocityY = -12
          if (!platform.isDisappearing) {
            platform.isDisappearing = true
            createParticles(player.x + player.width / 2, player.y + player.height, '#FF8080', 10)
          }
          break
          
        case PLATFORM_TYPES.BONUS:
          player.velocityY = -12
          const bigBonus = 50 + Math.floor(Math.random() * 100)
          score.value += bigBonus
          createParticles(player.x + player.width / 2, player.y + player.height, '#FFD700', 25)
          break
          
        default:
          player.velocityY = -12
      }
      
      break
    }
  }
  
  if (player.standingPlatform && player.standingPlatform.velocityX !== 0) {
    player.x += player.standingPlatform.velocityX
  }
  
  player.y += player.velocityY
  
  if (player.y < highestPlayerY) {
    highestPlayerY = player.y
  }
}

const updatePlatforms = () => {
  const targetCameraY = Math.min(cameraY, player.y - canvas.height * 0.4)
  if (targetCameraY < cameraY) {
    cameraY = targetCameraY
  }
  
  platforms.forEach(platform => {
    if (platform.velocityX !== 0) {
      platform.x += platform.velocityX
      if (platform.x <= 0) {
        platform.x = 0
        platform.velocityX *= -1
      }
      if (platform.x + platform.width >= canvas.width) {
        platform.x = canvas.width - platform.width
        platform.velocityX *= -1
      }
    }
    
    if (platform.isDisappearing) {
      platform.opacity -= 0.025
    }
  })
  
  platforms = platforms.filter(p => {
    const screenY = p.y - cameraY
    return screenY < canvas.height + 150 && p.opacity > 0
  })
  
  let highestPlatformY = Math.min(...platforms.map(p => p.y), Infinity)
  
  while (highestPlatformY > cameraY - difficultyParams.platformSpacing * 2) {
    const newY = highestPlatformY - difficultyParams.platformSpacing
    platforms.push(createPlatform(newY))
    highestPlatformY = newY
  }
}

const drawBackground = () => {
  const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height)
  gradient.addColorStop(0, '#E8F4F8')
  gradient.addColorStop(0.5, '#D4E5ED')
  gradient.addColorStop(1, '#C5DDE8')
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, canvas.width, canvas.height)
  
  const time = performance.now() / 1000
  for (let i = 0; i < 5; i++) {
    const x = (Math.sin(time * 0.2 + i) + 1) / 2 * canvas.width
    const y = (Math.cos(time * 0.15 + i * 1.5) + 1) / 2 * canvas.height
    const radius = 50 + Math.sin(time * 0.3 + i) * 20
    
    const glow = ctx.createRadialGradient(x, y, 0, x, y, radius)
    glow.addColorStop(0, 'rgba(255, 255, 255, 0.15)')
    glow.addColorStop(1, 'rgba(255, 255, 255, 0)')
    
    ctx.fillStyle = glow
    ctx.beginPath()
    ctx.arc(x, y, radius, 0, Math.PI * 2)
    ctx.fill()
  }
}

const drawPlatform = (platform) => {
  if (platform.opacity <= 0) return
  
  ctx.save()
  ctx.globalAlpha = platform.opacity
  
  const colors = COLORS.platforms[platform.type]
  
  ctx.shadowColor = colors.glow
  ctx.shadowBlur = 12
  ctx.shadowOffsetY = 4
  
  const radius = 9
  const x = platform.x
  const y = platform.y
  const w = platform.width
  const h = platform.height
  
  ctx.beginPath()
  if (ctx.roundRect) {
    ctx.roundRect(x, y, w, h, radius)
  } else {
    const r = radius
    ctx.moveTo(x + r, y)
    ctx.lineTo(x + w - r, y)
    ctx.quadraticCurveTo(x + w, y, x + w, y + r)
    ctx.lineTo(x + w, y + h - r)
    ctx.quadraticCurveTo(x + w, y + h, x + w - r, y + h)
    ctx.lineTo(x + r, y + h)
    ctx.quadraticCurveTo(x, y + h, x, y + h - r)
    ctx.lineTo(x, y + r)
    ctx.quadraticCurveTo(x, y, x + r, y)
  }
  
  const gradient = ctx.createLinearGradient(x, y, x, y + h)
  gradient.addColorStop(0, colors.top)
  gradient.addColorStop(1, colors.bottom)
  ctx.fillStyle = gradient
  ctx.fill()
  
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.5)'
  ctx.lineWidth = 1.5
  ctx.stroke()
  
  ctx.shadowColor = 'transparent'
  ctx.fillStyle = 'rgba(255, 255, 255, 0.3)'
  ctx.fillRect(x + 5, y + 3, w - 10, 3)
  
  if (platform.type === PLATFORM_TYPES.BOUNCY) {
    const springY = y + h / 2
    ctx.strokeStyle = 'rgba(100, 180, 255, 0.8)'
    ctx.lineWidth = 2.5
    ctx.beginPath()
    const springCount = Math.max(2, Math.floor(w / 50))
    for (let i = 0; i < springCount; i++) {
      const sx = x + 15 + i * (w - 30) / Math.max(1, springCount - 1)
      ctx.moveTo(sx - 7, springY)
      ctx.quadraticCurveTo(sx - 3, springY - 4, sx, springY)
      ctx.quadraticCurveTo(sx + 3, springY + 4, sx + 7, springY)
    }
    ctx.stroke()
  } else if (platform.type === PLATFORM_TYPES.BONUS) {
    const starX = x + w / 2
    const starY = y + h / 2
    ctx.fillStyle = 'rgba(255, 255, 255, 0.8)'
    drawStar(starX, starY, 5, 7, 3.5)
  } else if (platform.type === PLATFORM_TYPES.DISAPPEARING && platform.isDisappearing) {
    ctx.setLineDash([4, 4])
    ctx.strokeStyle = 'rgba(255, 50, 50, 0.7)'
    ctx.lineWidth = 2
    ctx.strokeRect(x - 2, y - 2, w + 4, h + 4)
    ctx.setLineDash([])
  }
  
  ctx.restore()
}

const drawStar = (cx, cy, spikes, outerRadius, innerRadius) => {
  let rot = Math.PI / 2 * 3
  let x = cx
  let y = cy
  const step = Math.PI / spikes
  
  ctx.beginPath()
  ctx.moveTo(cx, cy - outerRadius)
  
  for (let i = 0; i < spikes; i++) {
    x = cx + Math.cos(rot) * outerRadius
    y = cy + Math.sin(rot) * outerRadius
    ctx.lineTo(x, y)
    rot += step
    
    x = cx + Math.cos(rot) * innerRadius
    y = cy + Math.sin(rot) * innerRadius
    ctx.lineTo(x, y)
    rot += step
  }
  
  ctx.lineTo(cx, cy - outerRadius)
  ctx.closePath()
  ctx.fill()
}

const drawPlayer = () => {
  ctx.save()
  
  const x = player.x
  const y = player.y
  const w = player.width
  const h = player.height
  
  ctx.translate(x + w / 2, y + h / 2)
  
  if (!player.facingRight) {
    ctx.scale(-1, 1)
  }
  
  if (Math.abs(player.velocityX) > 0.3) {
    const tilt = Math.sin(player.animFrame) * 0.08
    ctx.rotate(tilt)
  }
  
  ctx.shadowColor = 'rgba(255, 105, 180, 0.35)'
  ctx.shadowBlur = 12
  ctx.shadowOffsetY = 4
  
  const bodyGradient = ctx.createRadialGradient(0, 5, 0, 0, 5, w / 2)
  bodyGradient.addColorStop(0, COLORS.player.body)
  bodyGradient.addColorStop(1, COLORS.player.bodyShadow)
  
  ctx.fillStyle = bodyGradient
  ctx.beginPath()
  ctx.ellipse(0, 8, w / 2 - 2, h / 2 - 5, 0, 0, Math.PI * 2)
  ctx.fill()
  
  ctx.shadowColor = 'transparent'
  
  ctx.strokeStyle = COLORS.player.outline
  ctx.lineWidth = 1.5
  ctx.stroke()
  
  ctx.fillStyle = 'rgba(255, 255, 255, 0.25)'
  ctx.beginPath()
  ctx.ellipse(-4, -4, 7, 4, -0.3, 0, Math.PI * 2)
  ctx.fill()
  
  ctx.fillStyle = COLORS.player.face
  ctx.beginPath()
  ctx.ellipse(0, 0, w / 2 - 4, h / 2 - 8, 0, 0, Math.PI * 2)
  ctx.fill()
  
  const eyeY = -3
  const eyeSpacing = 7
  
  ctx.fillStyle = 'white'
  ctx.beginPath()
  ctx.ellipse(-eyeSpacing, eyeY, 5.5, 6.5, 0, 0, Math.PI * 2)
  ctx.ellipse(eyeSpacing, eyeY, 5.5, 6.5, 0, 0, Math.PI * 2)
  ctx.fill()
  
  const lookX = player.velocityX * 0.25
  const lookY = player.velocityY > 3 ? 1 : 0
  
  ctx.fillStyle = COLORS.player.eyes
  ctx.beginPath()
  ctx.arc(-eyeSpacing + lookX, eyeY + lookY, 2.5, 0, Math.PI * 2)
  ctx.arc(eyeSpacing + lookX, eyeY + lookY, 2.5, 0, Math.PI * 2)
  ctx.fill()
  
  ctx.fillStyle = 'white'
  ctx.beginPath()
  ctx.arc(-eyeSpacing + lookX - 0.8, eyeY + lookY - 0.8, 1.2, 0, Math.PI * 2)
  ctx.arc(eyeSpacing + lookX - 0.8, eyeY + lookY - 0.8, 1.2, 0, Math.PI * 2)
  ctx.fill()
  
  ctx.fillStyle = 'rgba(255, 182, 193, 0.55)'
  ctx.beginPath()
  ctx.ellipse(-eyeSpacing - 7, eyeY + 5, 4.5, 2.8, 0, 0, Math.PI * 2)
  ctx.ellipse(eyeSpacing + 7, eyeY + 5, 4.5, 2.8, 0, 0, Math.PI * 2)
  ctx.fill()
  
  ctx.strokeStyle = '#FF69B4'
  ctx.lineWidth = 1.2
  ctx.lineCap = 'round'
  ctx.beginPath()
  if (player.velocityY > 4) {
    ctx.ellipse(0, 8, 3.5, 4.5, 0, 0, Math.PI * 2)
    ctx.stroke()
  } else {
    ctx.arc(0, 8, 3.5, 0, Math.PI)
    ctx.stroke()
  }
  
  ctx.restore()
}

const drawParticles = () => {
  particles.forEach(p => {
    ctx.save()
    ctx.globalAlpha = p.life
    ctx.fillStyle = p.color
    ctx.beginPath()
    ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2)
    ctx.fill()
    ctx.restore()
  })
}

const render = () => {
  ctx.clearRect(0, 0, canvas.width, canvas.height)
  
  drawBackground()
  
  ctx.save()
  ctx.translate(0, -cameraY)
  
  platforms.forEach(platform => drawPlatform(platform))
  
  drawParticles()
  
  if (gameState.value === 'playing' || gameState.value === 'paused') {
    drawPlayer()
  }
  
  ctx.restore()
}

const gameLoop = (currentTime = 0) => {
  if (gameState.value !== 'playing') return
  
  const dt = Math.min((currentTime - lastTime) / 16.67, 3)
  lastTime = currentTime
  
  updatePlayer(dt)
  handlePlatformCollision()
  updatePlatforms()
  updateParticles()
  
  if (checkGameOver()) {
    gameOver()
    return
  }
  
  render()
  
  animationId = requestAnimationFrame(gameLoop)
}

const handleResize = () => {
  resizeCanvas()
}

onMounted(() => {
  loadHighScore()
  initCanvas()
  initPlatforms()
  
  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('keyup', handleKeyUp)
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
  window.removeEventListener('keydown', handleKeyDown)
  window.removeEventListener('keyup', handleKeyUp)
  window.removeEventListener('resize', handleResize)
})
</script>

<style scoped>
.game-container {
  width: 100vw;
  height: 100vh;
  position: relative;
  overflow: hidden;
  background: linear-gradient(135deg, #E8F4F8 0%, #D4E5ED 50%, #C5DDE8 100%);
}

.background-animate {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  overflow: hidden;
  z-index: 1;
}

.cloud {
  position: absolute;
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.8) 0%, rgba(255, 255, 255, 0.4) 100%);
  border-radius: 50%;
  animation: floatCloud linear infinite;
  opacity: 0.6;
}

.cloud::before,
.cloud::after {
  content: '';
  position: absolute;
  background: inherit;
  border-radius: 50%;
}

.cloud::before {
  width: 60%;
  height: 80%;
  top: -30%;
  left: 20%;
}

.cloud::after {
  width: 50%;
  height: 70%;
  top: -20%;
  right: 10%;
}

@keyframes floatCloud {
  0% {
    transform: translateX(-100%);
  }
  100% {
    transform: translateX(calc(100vw + 100%));
  }
}

.game-canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 2;
}

.ui-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  padding: 20px;
  z-index: 10;
  pointer-events: none;
}

.score-panel {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 20px;
  padding: 15px 30px;
  margin: 0 auto;
  max-width: 400px;
  pointer-events: auto;
}

.score-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
}

.score-label {
  font-size: 12px;
  color: rgba(255, 255, 255, 0.7);
  text-transform: uppercase;
  letter-spacing: 1px;
  font-weight: 500;
}

.score-value {
  font-size: 24px;
  font-weight: 700;
  color: white;
  text-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
}

.score-value.highlight {
  color: #FFD700;
  text-shadow: 0 0 20px rgba(255, 215, 0, 0.5);
}

.score-divider {
  width: 1px;
  height: 40px;
  background: linear-gradient(180deg, transparent, rgba(255, 255, 255, 0.5), transparent);
}

.control-buttons {
  position: absolute;
  top: 20px;
  right: 20px;
  display: flex;
  gap: 10px;
  pointer-events: auto;
}

.glass-panel {
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.3);
  box-shadow: 
    0 8px 32px rgba(0, 0, 0, 0.1),
    inset 0 1px 0 rgba(255, 255, 255, 0.4);
}

.glass-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 20px;
  background: rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 15px;
  color: white;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.glass-btn:hover {
  background: rgba(255, 255, 255, 0.3);
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.15);
}

.glass-btn:active {
  transform: translateY(0);
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.glass-btn.primary {
  background: linear-gradient(135deg, rgba(255, 182, 193, 0.4) 0%, rgba(255, 105, 180, 0.4) 100%);
  border-color: rgba(255, 182, 193, 0.5);
}

.glass-btn.primary:hover {
  background: linear-gradient(135deg, rgba(255, 182, 193, 0.5) 0%, rgba(255, 105, 180, 0.5) 100%);
}

.btn-icon {
  font-size: 18px;
}

.btn-text {
  letter-spacing: 0.5px;
}

.start-screen,
.pause-screen,
.gameover-screen {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 20;
  background: rgba(0, 0, 0, 0.3);
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
}

.start-panel,
.pause-panel,
.gameover-panel {
  padding: 40px 50px;
  text-align: center;
  min-width: 300px;
  animation: slideUp 0.5s ease-out;
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.game-title {
  font-size: 32px;
  font-weight: 800;
  color: white;
  text-shadow: 
    0 4px 20px rgba(0, 0, 0, 0.3),
    0 0 40px rgba(255, 182, 193, 0.5);
  margin-bottom: 10px;
  background: linear-gradient(135deg, #FFB6C1, #FF69B4, #FFD700);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.game-subtitle {
  font-size: 16px;
  color: rgba(255, 255, 255, 0.8);
  margin-bottom: 20px;
  font-weight: 500;
}

.high-score-display {
  margin-bottom: 30px;
  padding: 15px;
  background: rgba(255, 215, 0, 0.1);
  border-radius: 10px;
  border: 1px solid rgba(255, 215, 0, 0.3);
}

.high-score-label {
  font-size: 14px;
  color: rgba(255, 255, 255, 0.7);
}

.high-score-value {
  font-size: 24px;
  font-weight: 700;
  color: #FFD700;
  margin-left: 10px;
  text-shadow: 0 0 20px rgba(255, 215, 0, 0.5);
}

.start-btn {
  padding: 18px 40px;
  font-size: 18px;
  justify-content: center;
  margin-bottom: 25px;
  background: linear-gradient(135deg, rgba(255, 182, 193, 0.4) 0%, rgba(255, 105, 180, 0.4) 100%);
}

.start-btn:hover {
  background: linear-gradient(135deg, rgba(255, 182, 193, 0.5) 0%, rgba(255, 105, 180, 0.5) 100%);
  transform: translateY(-3px) scale(1.02);
}

.controls-hint {
  font-size: 13px;
  color: rgba(255, 255, 255, 0.6);
  line-height: 1.8;
}

.controls-hint p {
  margin: 5px 0;
}

.pause-title,
.gameover-title {
  font-size: 28px;
  font-weight: 700;
  color: white;
  margin-bottom: 25px;
  text-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
}

.pause-stats {
  margin-bottom: 30px;
  padding: 20px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 15px;
}

.pause-stats p {
  font-size: 16px;
  color: rgba(255, 255, 255, 0.8);
  margin: 10px 0;
}

.pause-buttons {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.pause-buttons .control-btn {
  justify-content: center;
  padding: 15px 25px;
}

.gameover-stats {
  margin-bottom: 30px;
}

.stat-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 20px;
  margin: 8px 0;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 12px;
}

.stat-label {
  font-size: 15px;
  color: rgba(255, 255, 255, 0.8);
  font-weight: 500;
}

.stat-label.new-record {
  font-size: 16px;
  color: #FFD700;
  font-weight: 700;
}

.stat-value {
  font-size: 20px;
  font-weight: 700;
  color: white;
}

.stat-value.highlight {
  color: #FFD700;
  text-shadow: 0 0 20px rgba(255, 215, 0, 0.5);
}

.restart-btn {
  padding: 16px 40px;
  font-size: 16px;
  justify-content: center;
  width: 100%;
}

.touch-areas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  z-index: 5;
  pointer-events: auto;
}

.touch-area {
  flex: 1;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  padding-bottom: 100px;
  opacity: 0;
  transition: opacity 0.3s;
}

.touch-area:active {
  opacity: 0.3;
  background: rgba(255, 255, 255, 0.2);
}

.touch-area.left::after {
  content: '◀';
  font-size: 40px;
  color: rgba(255, 255, 255, 0.5);
}

.touch-area.right::after {
  content: '▶';
  font-size: 40px;
  color: rgba(255, 255, 255, 0.5);
}

.highlight {
  color: #FFD700;
}

@media (max-width: 600px) {
  .score-panel {
    padding: 12px 20px;
    gap: 15px;
  }
  
  .score-value {
    font-size: 20px;
  }
  
  .control-buttons {
    top: auto;
    bottom: 20px;
    right: 20px;
  }
  
  .glass-btn {
    padding: 10px 15px;
    font-size: 12px;
  }
  
  .start-panel,
  .pause-panel,
  .gameover-panel {
    padding: 30px 25px;
    min-width: 280px;
    margin: 20px;
  }
  
  .game-title {
    font-size: 26px;
  }
  
  .start-btn {
    padding: 15px 30px;
    font-size: 16px;
  }
}

@media (max-height: 600px) {
  .ui-overlay {
    padding: 10px;
  }
  
  .score-panel {
    padding: 10px 15px;
  }
  
  .control-buttons {
    top: 10px;
    right: 10px;
  }
  
  .glass-btn {
    padding: 8px 12px;
    font-size: 11px;
  }
}
</style>
