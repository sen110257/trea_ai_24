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
import { ref, onMounted, onUnmounted, computed } from 'vue'

const gameCanvas = ref(null)
const canvas = ref(null)
const ctx = ref(null)

const gameState = ref('idle')
const score = ref(0)
const currentFloor = ref(0)
const highScore = ref(0)
const isNewHighScore = ref(false)

let animationId = null
let lastTime = 0
let difficulty = 1

const player = {
  x: 0,
  y: 0,
  width: 40,
  height: 50,
  velocityX: 0,
  velocityY: 0,
  maxSpeed: 8,
  acceleration: 0.8,
  friction: 0.9,
  facingRight: true,
  isOnPlatform: false,
  currentPlatform: null,
  animFrame: 0
}

let platforms = []
let particles = []

const keys = {
  left: false,
  right: false
}

const touchDirection = ref(null)

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
  },
  background: {
    top: '#E8F4F8',
    bottom: '#D4E5ED'
  }
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

const initCanvas = () => {
  canvas.value = gameCanvas.value
  ctx.value = canvas.value.getContext('2d')
  resizeCanvas()
}

const resizeCanvas = () => {
  const container = canvas.value.parentElement
  canvas.value.width = container.clientWidth
  canvas.value.height = container.clientHeight
  
  if (gameState.value === 'idle') {
    player.x = canvas.value.width / 2 - player.width / 2
    player.y = 100
  }
}

const createPlatform = (y, type = PLATFORM_TYPES.NORMAL) => {
  const baseWidth = Math.max(80, 150 - difficulty * 5)
  const width = baseWidth + Math.random() * 40
  
  return {
    x: Math.random() * (canvas.value.width - width),
    y: y,
    width: width,
    height: 20,
    type: type,
    opacity: 1,
    isDisappearing: false,
    disappearTimer: 0,
    velocityX: Math.random() > 0.7 ? (Math.random() - 0.5) * (2 + difficulty * 0.3) : 0,
    touched: false
  }
}

const initPlatforms = () => {
  platforms = []
  const startPlatform = {
    x: canvas.value.width / 2 - 80,
    y: 150,
    width: 160,
    height: 20,
    type: PLATFORM_TYPES.NORMAL,
    opacity: 1,
    isDisappearing: false,
    disappearTimer: 0,
    velocityX: 0,
    touched: false
  }
  platforms.push(startPlatform)
  
  let y = 250
  const spacing = 100
  
  while (y < canvas.value.height + 200) {
    const type = getRandomPlatformType()
    platforms.push(createPlatform(y, type))
    y += spacing
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

const initPlayer = () => {
  player.x = canvas.value.width / 2 - player.width / 2
  player.y = 80
  player.velocityX = 0
  player.velocityY = 0
  player.isOnPlatform = true
  player.currentPlatform = platforms[0]
}

const startGame = () => {
  gameState.value = 'playing'
  score.value = 0
  currentFloor.value = 0
  difficulty = 1
  isNewHighScore.value = false
  particles = []
  
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
  touchDirection.value = direction
  if (direction === 'left') {
    keys.left = true
    keys.right = false
  } else {
    keys.right = true
    keys.left = false
  }
}

const handleTouchEnd = () => {
  touchDirection.value = null
  keys.left = false
  keys.right = false
}

const createParticles = (x, y, color, count = 10) => {
  for (let i = 0; i < count; i++) {
    particles.push({
      x: x,
      y: y,
      velocityX: (Math.random() - 0.5) * 8,
      velocityY: (Math.random() - 0.5) * 8 - 2,
      size: 3 + Math.random() * 5,
      color: color,
      life: 1,
      decay: 0.02 + Math.random() * 0.02
    })
  }
}

const updateParticles = () => {
  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i]
    p.x += p.velocityX
    p.y += p.velocityY
    p.velocityY += 0.2
    p.life -= p.decay
    
    if (p.life <= 0) {
      particles.splice(i, 1)
    }
  }
}

const drawParticles = () => {
  particles.forEach(p => {
    ctx.value.save()
    ctx.value.globalAlpha = p.life
    ctx.value.fillStyle = p.color
    ctx.value.beginPath()
    ctx.value.arc(p.x, p.y, p.size, 0, Math.PI * 2)
    ctx.value.fill()
    ctx.value.restore()
  })
}

const updateDifficulty = () => {
  difficulty = 1 + Math.floor(currentFloor.value / 10) * 0.5
  if (difficulty > 10) difficulty = 10
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
  
  if (!player.isOnPlatform) {
    player.velocityY += 0.5
  }
  
  player.x += player.velocityX
  player.y += player.velocityY
  
  if (player.x + player.width < 0) {
    player.x = canvas.value.width
  } else if (player.x > canvas.value.width) {
    player.x = -player.width
  }
  
  if (player.velocityX !== 0) {
    player.animFrame += 0.15
  }
  
  player.isOnPlatform = false
  player.currentPlatform = null
}

const checkPlatformCollision = () => {
  for (const platform of platforms) {
    if (platform.isDisappearing && platform.opacity < 0.3) continue
    
    const playerBottom = player.y + player.height
    const playerCenterX = player.x + player.width / 2
    
    if (player.velocityY > 0 &&
        playerBottom >= platform.y &&
        playerBottom <= platform.y + platform.height + player.velocityY &&
        playerCenterX >= platform.x &&
        playerCenterX <= platform.x + platform.width) {
      
      player.y = platform.y - player.height
      player.isOnPlatform = true
      player.currentPlatform = platform
      
      if (!platform.touched) {
        platform.touched = true
        currentFloor.value++
        updateDifficulty()
        
        const baseScore = 10
        const bonusScore = Math.floor(Math.random() * 20)
        score.value += baseScore + bonusScore
        createParticles(player.x + player.width / 2, player.y + player.height, '#FFD700', 5)
      }
      
      switch (platform.type) {
        case PLATFORM_TYPES.BOUNCY:
          player.velocityY = -12
          player.isOnPlatform = false
          createParticles(player.x + player.width / 2, player.y + player.height, '#80C8FF', 15)
          break
          
        case PLATFORM_TYPES.DISAPPEARING:
          platform.isDisappearing = true
          createParticles(player.x + player.width / 2, player.y + player.height, '#FF8080', 10)
          break
          
        case PLATFORM_TYPES.BONUS:
          const bigBonus = 50 + Math.floor(Math.random() * 100)
          score.value += bigBonus
          createParticles(player.x + player.width / 2, player.y + player.height, '#FFD700', 25)
          break
          
        default:
          player.velocityY = 0
      }
      
      if (platform.velocityX !== 0 && platform.type !== PLATFORM_TYPES.BOUNCY) {
        player.x += platform.velocityX
      }
      
      break
    }
  }
}

const updatePlatforms = (dt) => {
  const scrollSpeed = 1 + difficulty * 0.3
  const targetY = canvas.value.height * 0.4
  
  if (player.y < targetY && player.velocityY < 0) {
    const scrollAmount = targetY - player.y
    player.y = targetY
    
    platforms.forEach(p => {
      p.y += scrollAmount
    })
  }
  
  platforms.forEach(platform => {
    if (platform.velocityX !== 0) {
      platform.x += platform.velocityX
      if (platform.x <= 0 || platform.x + platform.width >= canvas.value.width) {
        platform.velocityX *= -1
      }
    }
    
    if (platform.isDisappearing) {
      platform.opacity -= 0.03
    }
  })
  
  platforms = platforms.filter(p => p.y < canvas.value.height + 100 && p.opacity > 0)
  
  let highestPlatformY = Math.min(...platforms.map(p => p.y))
  const spacing = Math.max(60, 100 - difficulty * 3)
  
  while (highestPlatformY > -spacing) {
    const type = getRandomPlatformType()
    platforms.push(createPlatform(highestPlatformY - spacing, type))
    highestPlatformY -= spacing
  }
  
  if (player.y > canvas.value.height + 50) {
    gameOver()
  }
}

const drawBackground = () => {
  const gradient = ctx.value.createLinearGradient(0, 0, 0, canvas.value.height)
  gradient.addColorStop(0, '#E8F4F8')
  gradient.addColorStop(0.5, '#D4E5ED')
  gradient.addColorStop(1, '#C5DDE8')
  ctx.value.fillStyle = gradient
  ctx.value.fillRect(0, 0, canvas.value.width, canvas.value.height)
  
  const time = performance.now() / 1000
  for (let i = 0; i < 5; i++) {
    const x = (Math.sin(time * 0.2 + i) + 1) / 2 * canvas.value.width
    const y = (Math.cos(time * 0.15 + i * 1.5) + 1) / 2 * canvas.value.height
    const radius = 50 + Math.sin(time * 0.3 + i) * 20
    
    const glow = ctx.value.createRadialGradient(x, y, 0, x, y, radius)
    glow.addColorStop(0, 'rgba(255, 255, 255, 0.15)')
    glow.addColorStop(1, 'rgba(255, 255, 255, 0)')
    
    ctx.value.fillStyle = glow
    ctx.value.beginPath()
    ctx.value.arc(x, y, radius, 0, Math.PI * 2)
    ctx.value.fill()
  }
}

const drawPlatform = (platform) => {
  if (platform.opacity <= 0) return
  
  ctx.value.save()
  ctx.value.globalAlpha = platform.opacity
  
  const colors = COLORS.platforms[platform.type]
  
  ctx.shadowColor = colors.glow
  ctx.shadowBlur = 15
  ctx.shadowOffsetY = 5
  
  const radius = 10
  const x = platform.x
  const y = platform.y
  const w = platform.width
  const h = platform.height
  
  ctx.value.beginPath()
  ctx.value.roundRect(x, y, w, h, radius)
  
  const gradient = ctx.value.createLinearGradient(x, y, x, y + h)
  gradient.addColorStop(0, colors.top)
  gradient.addColorStop(1, colors.bottom)
  ctx.value.fillStyle = gradient
  ctx.value.fill()
  
  ctx.value.strokeStyle = 'rgba(255, 255, 255, 0.5)'
  ctx.value.lineWidth = 2
  ctx.value.stroke()
  
  ctx.value.shadowColor = 'transparent'
  ctx.value.fillStyle = 'rgba(255, 255, 255, 0.3)'
  ctx.value.fillRect(x + 5, y + 3, w - 10, 4)
  
  if (platform.type === PLATFORM_TYPES.BOUNCY) {
    const springY = y + h / 2
    ctx.value.strokeStyle = 'rgba(100, 180, 255, 0.8)'
    ctx.value.lineWidth = 3
    ctx.value.beginPath()
    for (let i = 0; i < 4; i++) {
      const sx = x + 15 + i * (w - 30) / 3
      ctx.value.moveTo(sx - 8, springY)
      ctx.value.quadraticCurveTo(sx - 4, springY - 5, sx, springY)
      ctx.value.quadraticCurveTo(sx + 4, springY + 5, sx + 8, springY)
    }
    ctx.value.stroke()
  } else if (platform.type === PLATFORM_TYPES.BONUS) {
    const starX = x + w / 2
    const starY = y + h / 2
    ctx.value.fillStyle = 'rgba(255, 255, 255, 0.8)'
    drawStar(starX, starY, 5, 8, 4)
  } else if (platform.type === PLATFORM_TYPES.DISAPPEARING && platform.isDisappearing) {
    ctx.value.fillStyle = 'rgba(255, 100, 100, 0.3)'
    ctx.value.setLineDash([5, 5])
    ctx.value.strokeStyle = 'rgba(255, 50, 50, 0.8)'
    ctx.value.lineWidth = 2
    ctx.value.strokeRect(x - 2, y - 2, w + 4, h + 4)
    ctx.value.setLineDash([])
  }
  
  ctx.value.restore()
}

const drawStar = (cx, cy, spikes, outerRadius, innerRadius) => {
  let rot = Math.PI / 2 * 3
  let x = cx
  let y = cy
  const step = Math.PI / spikes
  
  ctx.value.beginPath()
  ctx.value.moveTo(cx, cy - outerRadius)
  
  for (let i = 0; i < spikes; i++) {
    x = cx + Math.cos(rot) * outerRadius
    y = cy + Math.sin(rot) * outerRadius
    ctx.value.lineTo(x, y)
    rot += step
    
    x = cx + Math.cos(rot) * innerRadius
    y = cy + Math.sin(rot) * innerRadius
    ctx.value.lineTo(x, y)
    rot += step
  }
  
  ctx.value.lineTo(cx, cy - outerRadius)
  ctx.value.closePath()
  ctx.value.fill()
}

const drawPlayer = () => {
  ctx.value.save()
  
  const x = player.x
  const y = player.y
  const w = player.width
  const h = player.height
  
  ctx.value.translate(x + w / 2, y + h / 2)
  
  if (!player.facingRight) {
    ctx.value.scale(-1, 1)
  }
  
  if (Math.abs(player.velocityX) > 0.5) {
    const tilt = Math.sin(player.animFrame) * 0.1
    ctx.value.rotate(tilt)
  }
  
  ctx.value.shadowColor = 'rgba(255, 105, 180, 0.4)'
  ctx.value.shadowBlur = 15
  ctx.value.shadowOffsetY = 5
  
  const bodyGradient = ctx.value.createRadialGradient(0, 5, 0, 0, 5, w / 2)
  bodyGradient.addColorStop(0, COLORS.player.body)
  bodyGradient.addColorStop(1, COLORS.player.bodyShadow)
  
  ctx.value.fillStyle = bodyGradient
  ctx.value.beginPath()
  ctx.value.ellipse(0, 8, w / 2 - 2, h / 2 - 5, 0, 0, Math.PI * 2)
  ctx.value.fill()
  
  ctx.value.shadowColor = 'transparent'
  
  ctx.value.strokeStyle = COLORS.player.outline
  ctx.value.lineWidth = 2
  ctx.value.stroke()
  
  ctx.value.fillStyle = 'rgba(255, 255, 255, 0.3)'
  ctx.value.beginPath()
  ctx.value.ellipse(-5, -5, 8, 5, -0.3, 0, Math.PI * 2)
  ctx.value.fill()
  
  ctx.value.fillStyle = COLORS.player.face
  ctx.value.beginPath()
  ctx.value.ellipse(0, 0, w / 2 - 4, h / 2 - 8, 0, 0, Math.PI * 2)
  ctx.value.fill()
  
  const eyeY = -3
  const eyeSpacing = 8
  
  ctx.value.fillStyle = 'white'
  ctx.value.beginPath()
  ctx.value.ellipse(-eyeSpacing, eyeY, 6, 7, 0, 0, Math.PI * 2)
  ctx.value.ellipse(eyeSpacing, eyeY, 6, 7, 0, 0, Math.PI * 2)
  ctx.value.fill()
  
  const lookX = player.velocityX * 0.3
  const lookY = player.velocityY > 0 ? 1 : 0
  
  ctx.value.fillStyle = COLORS.player.eyes
  ctx.value.beginPath()
  ctx.value.arc(-eyeSpacing + lookX, eyeY + lookY, 3, 0, Math.PI * 2)
  ctx.value.arc(eyeSpacing + lookX, eyeY + lookY, 3, 0, Math.PI * 2)
  ctx.value.fill()
  
  ctx.value.fillStyle = 'white'
  ctx.value.beginPath()
  ctx.value.arc(-eyeSpacing + lookX - 1, eyeY + lookY - 1, 1.5, 0, Math.PI * 2)
  ctx.value.arc(eyeSpacing + lookX - 1, eyeY + lookY - 1, 1.5, 0, Math.PI * 2)
  ctx.value.fill()
  
  ctx.value.fillStyle = 'rgba(255, 182, 193, 0.6)'
  ctx.value.beginPath()
  ctx.value.ellipse(-eyeSpacing - 8, eyeY + 5, 5, 3, 0, 0, Math.PI * 2)
  ctx.value.ellipse(eyeSpacing + 8, eyeY + 5, 5, 3, 0, 0, Math.PI * 2)
  ctx.value.fill()
  
  ctx.value.strokeStyle = '#FF69B4'
  ctx.value.lineWidth = 1.5
  ctx.value.lineCap = 'round'
  ctx.value.beginPath()
  if (player.velocityY > 5) {
    ctx.value.ellipse(0, 8, 4, 5, 0, 0, Math.PI * 2)
    ctx.value.stroke()
  } else {
    ctx.value.arc(0, 8, 4, 0, Math.PI)
    ctx.value.stroke()
  }
  
  ctx.value.restore()
}

const render = () => {
  ctx.value.clearRect(0, 0, canvas.value.width, canvas.value.height)
  
  drawBackground()
  
  platforms.forEach(platform => drawPlatform(platform))
  
  drawParticles()
  
  if (gameState.value === 'playing' || gameState.value === 'paused') {
    drawPlayer()
  }
}

const gameLoop = (currentTime = 0) => {
  if (gameState.value !== 'playing') return
  
  const dt = (currentTime - lastTime) / 16.67
  lastTime = currentTime
  
  updatePlayer(dt)
  checkPlatformCollision()
  updatePlatforms(dt)
  updateParticles()
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
