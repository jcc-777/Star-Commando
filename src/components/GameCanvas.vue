<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'

type GamePhase = 'ready' | 'running' | 'paused' | 'levelup' | 'gameover'

type Bullet = {
  x: number
  y: number
  width: number
  height: number
  speed: number
}

type Enemy = {
  x: number
  y: number
  width: number
  height: number
  speed: number
  hp: number
  maxHp: number
  score: number
  elite: boolean
  phase: number
  drift: number
}

type Particle = {
  x: number
  y: number
  vx: number
  vy: number
  life: number
  maxLife: number
  size: number
  color: string
}

type Star = {
  x: number
  y: number
  size: number
  speed: number
  alpha: number
}

type Player = {
  x: number
  y: number
  width: number
  height: number
  speed: number
  lives: number
  cooldown: number
  invincible: number
}

const props = defineProps<{
  phase: GamePhase
  restartToken: number
}>()

const emit = defineEmits<{
  'stats-change': [{ score: number; level: number; lives: number; eliteChance: number }]
  'level-clear': [{ clearedLevel: number; nextLevel: number }]
  'game-over': [{ score: number }]
}>()

const canvasRef = ref<HTMLCanvasElement | null>(null)
const wrapperRef = ref<HTMLDivElement | null>(null)

const baseWidth = 450
const baseHeight = 800
const player = ref<Player>({
  x: baseWidth / 2 - 22,
  y: baseHeight - 120,
  width: 44,
  height: 64,
  speed: 360,
  lives: 3,
  cooldown: 0,
  invincible: 0,
})

const bullets = ref<Bullet[]>([])
const enemies = ref<Enemy[]>([])
const particles = ref<Particle[]>([])
const farStars = ref<Star[]>([])
const nearStars = ref<Star[]>([])
const keys = ref<Record<string, boolean>>({})
const score = ref(0)
const level = ref(1)
const lastLevelThreshold = ref(0)
const animationFrame = ref<number | null>(null)
const lastTimestamp = ref(0)
const enemySpawnTimer = ref(0)
const mobileFiring = ref(false)
const joystickActive = ref(false)
const joystickPointerId = ref<number | null>(null)
const firePointerId = ref<number | null>(null)
const joystickVector = ref({ x: 0, y: 0 })

const paletteList = [
  { top: '#02030a', bottom: '#09111f', accent: '#22d3ee' },
  { top: '#060816', bottom: '#171b4f', accent: '#818cf8' },
  { top: '#140718', bottom: '#35084d', accent: '#f472b6' },
  { top: '#041421', bottom: '#07293d', accent: '#2dd4bf' },
  { top: '#180710', bottom: '#1f1d3a', accent: '#f59e0b' },
]

const joystickCenter = computed(() => ({ x: 74, y: baseHeight - 84 }))
const fireButtonCenter = computed(() => ({ x: baseWidth - 74, y: baseHeight - 84 }))
const eliteChance = computed(() => Math.min(0.12 + (level.value - 1) * 0.05, 0.55))

const clamp = (value: number, min: number, max: number) => Math.min(Math.max(value, min), max)

const resetGameState = () => {
  player.value = {
    x: baseWidth / 2 - 22,
    y: baseHeight - 120,
    width: 44,
    height: 64,
    speed: 360,
    lives: 3,
    cooldown: 0,
    invincible: 0,
  }
  bullets.value = []
  enemies.value = []
  particles.value = []
  score.value = 0
  level.value = 1
  lastLevelThreshold.value = 0
  enemySpawnTimer.value = 0
  joystickVector.value = { x: 0, y: 0 }
  joystickActive.value = false
  joystickPointerId.value = null
  firePointerId.value = null
  mobileFiring.value = false
  initStars()
  emitStats()
}

const initStars = () => {
  farStars.value = Array.from({ length: 70 }, () => ({
    x: Math.random() * baseWidth,
    y: Math.random() * baseHeight,
    size: Math.random() * 1.5 + 0.6,
    speed: Math.random() * 30 + 18,
    alpha: Math.random() * 0.4 + 0.2,
  }))

  nearStars.value = Array.from({ length: 38 }, () => ({
    x: Math.random() * baseWidth,
    y: Math.random() * baseHeight,
    size: Math.random() * 2.4 + 1.2,
    speed: Math.random() * 90 + 52,
    alpha: Math.random() * 0.5 + 0.35,
  }))
}

const emitStats = () => {
  emit('stats-change', {
    score: score.value,
    level: level.value,
    lives: player.value.lives,
    eliteChance: eliteChance.value,
  })
}

const resizeCanvas = () => {
  const canvas = canvasRef.value
  const wrapper = wrapperRef.value
  if (!canvas || !wrapper) {
    return
  }

  const dpr = window.devicePixelRatio || 1
  const rect = wrapper.getBoundingClientRect()
  canvas.width = Math.floor(rect.width * dpr)
  canvas.height = Math.floor(rect.height * dpr)
  canvas.style.width = `${rect.width}px`
  canvas.style.height = `${rect.height}px`

  const context = canvas.getContext('2d')
  if (!context) {
    return
  }

  context.setTransform(dpr, 0, 0, dpr, 0, 0)
}

const createParticles = (x: number, y: number, color: string, amount: number) => {
  for (let index = 0; index < amount; index += 1) {
    const angle = (Math.PI * 2 * index) / amount + Math.random() * 0.8
    const speed = Math.random() * 120 + 50
    particles.value.push({
      x,
      y,
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed,
      life: 0.5 + Math.random() * 0.35,
      maxLife: 0.5 + Math.random() * 0.35,
      size: Math.random() * 3.5 + 1.2,
      color,
    })
  }
}

const spawnBullet = () => {
  bullets.value.push({
    x: player.value.x + player.value.width / 2 - 3,
    y: player.value.y - 8,
    width: 6,
    height: 18,
    speed: 620,
  })
}

const spawnEnemy = () => {
  const elite = Math.random() < eliteChance.value
  const width = elite ? 54 : 38
  const height = elite ? 64 : 42
  const speedBase = 105 + (level.value - 1) * 18
  const speed = elite ? speedBase * 0.88 : speedBase + Math.random() * 24

  enemies.value.push({
    x: Math.random() * (baseWidth - width),
    y: -height - Math.random() * 80,
    width,
    height,
    speed,
    hp: elite ? 3 + Math.floor(level.value / 2) : 1,
    maxHp: elite ? 3 + Math.floor(level.value / 2) : 1,
    score: elite ? 180 + level.value * 20 : 100,
    elite,
    phase: Math.random() * Math.PI * 2,
    drift: (Math.random() - 0.5) * (elite ? 40 : 20),
  })
}

const fireIfNeeded = (delta: number) => {
  if (player.value.cooldown > 0) {
    player.value.cooldown -= delta
  }

  const wantsToFire = props.phase === 'running' && (keys.value.Space || mobileFiring.value)
  if (!wantsToFire || player.value.cooldown > 0) {
    return
  }

  spawnBullet()
  player.value.cooldown = 0.16
}

const updatePlayer = (delta: number) => {
  const keyboardX = (keys.value.ArrowRight ? 1 : 0) - (keys.value.ArrowLeft ? 1 : 0)
  const keyboardY = (keys.value.ArrowDown ? 1 : 0) - (keys.value.ArrowUp ? 1 : 0)
  const moveX = clamp(keyboardX + joystickVector.value.x, -1, 1)
  const moveY = clamp(keyboardY + joystickVector.value.y, -1, 1)
  const length = Math.hypot(moveX, moveY) || 1
  const normalizedX = length > 1 ? moveX / length : moveX
  const normalizedY = length > 1 ? moveY / length : moveY

  player.value.x = clamp(
    player.value.x + normalizedX * player.value.speed * delta,
    12,
    baseWidth - player.value.width - 12,
  )
  player.value.y = clamp(
    player.value.y + normalizedY * player.value.speed * delta,
    18,
    baseHeight - player.value.height - 18,
  )

  if (player.value.invincible > 0) {
    player.value.invincible -= delta
  }
}

const updateBullets = (delta: number) => {
  bullets.value = bullets.value
    .map((bullet) => ({ ...bullet, y: bullet.y - bullet.speed * delta }))
    .filter((bullet) => bullet.y + bullet.height > 0)
}

const updateEnemies = (delta: number) => {
  enemySpawnTimer.value -= delta
  const spawnInterval = Math.max(0.36, 1.15 - (level.value - 1) * 0.09)
  if (enemySpawnTimer.value <= 0) {
    spawnEnemy()
    enemySpawnTimer.value = spawnInterval
  }

  enemies.value = enemies.value
    .map((enemy) => {
      const verticalSpeed = enemy.speed + (level.value - 1) * 8
      return {
        ...enemy,
        y: enemy.y + verticalSpeed * delta,
        x: clamp(
          enemy.x + Math.sin(performance.now() / 500 + enemy.phase) * enemy.drift * delta,
          6,
          baseWidth - enemy.width - 6,
        ),
      }
    })
    .filter((enemy) => {
      if (enemy.y <= baseHeight + 80) {
        return true
      }

      createParticles(enemy.x + enemy.width / 2, baseHeight - 20, '#fb7185', 8)
      return false
    })
}

const intersects = (
  first: { x: number; y: number; width: number; height: number },
  second: { x: number; y: number; width: number; height: number },
) =>
  first.x < second.x + second.width &&
  first.x + first.width > second.x &&
  first.y < second.y + second.height &&
  first.y + first.height > second.y

const applyDamageToPlayer = () => {
  if (player.value.invincible > 0) {
    return
  }

  player.value.lives -= 1
  player.value.invincible = 1.3
  createParticles(
    player.value.x + player.value.width / 2,
    player.value.y + player.value.height / 2,
    '#22d3ee',
    18,
  )
  emitStats()

  if (player.value.lives <= 0) {
    emit('game-over', { score: score.value })
  }
}

const handleCollisions = () => {
  const bulletsToRemove = new Set<number>()
  const enemiesToRemove = new Set<number>()

  bullets.value.forEach((bullet, bulletIndex) => {
    enemies.value.forEach((enemy, enemyIndex) => {
      if (bulletsToRemove.has(bulletIndex) || enemiesToRemove.has(enemyIndex)) {
        return
      }

      if (!intersects(bullet, enemy)) {
        return
      }

      bulletsToRemove.add(bulletIndex)
      enemy.hp -= 1
      createParticles(bullet.x + bullet.width / 2, bullet.y, enemy.elite ? '#f472b6' : '#fde047', 7)

      if (enemy.hp <= 0) {
        enemiesToRemove.add(enemyIndex)
        score.value += enemy.score
        createParticles(enemy.x + enemy.width / 2, enemy.y + enemy.height / 2, enemy.elite ? '#f472b6' : '#22d3ee', enemy.elite ? 24 : 16)
      }
    })
  })

  bullets.value = bullets.value.filter((_, index) => !bulletsToRemove.has(index))
  enemies.value = enemies.value.filter((_, index) => !enemiesToRemove.has(index))

  enemies.value = enemies.value.filter((enemy) => {
    const hitPlayer = intersects(enemy, player.value)
    if (!hitPlayer) {
      return true
    }

    createParticles(enemy.x + enemy.width / 2, enemy.y + enemy.height / 2, '#fb7185', enemy.elite ? 18 : 12)
    applyDamageToPlayer()
    return false
  })

  const newThreshold = Math.floor(score.value / 500)
  if (newThreshold > lastLevelThreshold.value) {
    lastLevelThreshold.value = newThreshold
    level.value = newThreshold + 1
    emitStats()
    emit('level-clear', {
      clearedLevel: newThreshold,
      nextLevel: level.value,
    })
    return
  }

  emitStats()
}

const updateParticles = (delta: number) => {
  particles.value = particles.value
    .map((particle) => ({
      ...particle,
      x: particle.x + particle.vx * delta,
      y: particle.y + particle.vy * delta,
      vy: particle.vy + 10 * delta,
      life: particle.life - delta,
    }))
    .filter((particle) => particle.life > 0)
}

const updateStars = (delta: number) => {
  const driftMultiplier = 1 + (level.value - 1) * 0.15
  const moveGroup = (group: Star[]) =>
    group.map((star) => {
      const nextY = star.y + star.speed * driftMultiplier * delta
      return {
        ...star,
        y: nextY > baseHeight ? -10 : nextY,
        x: nextY > baseHeight ? Math.random() * baseWidth : star.x,
      }
    })

  farStars.value = moveGroup(farStars.value)
  nearStars.value = moveGroup(nearStars.value)
}

const drawBackground = (context: CanvasRenderingContext2D) => {
  const palette = paletteList[(level.value - 1) % paletteList.length]
  const gradient = context.createLinearGradient(0, 0, 0, baseHeight)
  gradient.addColorStop(0, palette.top)
  gradient.addColorStop(1, palette.bottom)
  context.fillStyle = gradient
  context.fillRect(0, 0, baseWidth, baseHeight)

  const horizon = context.createLinearGradient(0, 0, baseWidth, baseHeight)
  horizon.addColorStop(0, '#00000000')
  horizon.addColorStop(0.5, `${palette.accent}10`)
  horizon.addColorStop(1, '#00000000')
  context.fillStyle = horizon
  context.fillRect(0, 0, baseWidth, baseHeight)

  const burst = context.createRadialGradient(baseWidth / 2, baseHeight * 0.28, 10, baseWidth / 2, baseHeight * 0.28, baseWidth * 0.72)
  burst.addColorStop(0, `${palette.accent}33`)
  burst.addColorStop(1, '#00000000')
  context.fillStyle = burst
  context.fillRect(0, 0, baseWidth, baseHeight)

  context.save()
  context.strokeStyle = `${palette.accent}12`
  context.lineWidth = 1
  for (let x = 0; x <= baseWidth; x += 36) {
    context.beginPath()
    context.moveTo(x, 0)
    context.lineTo(x, baseHeight)
    context.stroke()
  }
  for (let y = 0; y <= baseHeight; y += 36) {
    context.beginPath()
    context.moveTo(0, y)
    context.lineTo(baseWidth, y)
    context.stroke()
  }
  context.restore()
}

const drawStars = (context: CanvasRenderingContext2D, group: Star[], color: string) => {
  group.forEach((star) => {
    context.save()
    context.globalAlpha = star.alpha
    context.fillStyle = color
    context.beginPath()
    context.arc(star.x, star.y, star.size, 0, Math.PI * 2)
    context.fill()
    context.restore()
  })
}

const drawPlayer = (context: CanvasRenderingContext2D) => {
  const blinking = player.value.invincible > 0 && Math.floor(player.value.invincible * 12) % 2 === 0
  if (blinking) {
    return
  }

  context.save()
  context.translate(player.value.x, player.value.y)

  const hullGradient = context.createLinearGradient(0, 0, 0, player.value.height)
  hullGradient.addColorStop(0, '#e0f2fe')
  hullGradient.addColorStop(0.5, '#38bdf8')
  hullGradient.addColorStop(1, '#1d4ed8')
  context.fillStyle = hullGradient

  context.beginPath()
  context.moveTo(player.value.width / 2, 0)
  context.lineTo(player.value.width - 4, player.value.height - 10)
  context.lineTo(player.value.width / 2 + 8, player.value.height - 20)
  context.lineTo(player.value.width / 2 + 4, player.value.height)
  context.lineTo(player.value.width / 2 - 4, player.value.height)
  context.lineTo(player.value.width / 2 - 8, player.value.height - 20)
  context.lineTo(4, player.value.height - 10)
  context.closePath()
  context.fill()

  context.fillStyle = '#7dd3fc'
  context.fillRect(player.value.width / 2 - 6, 18, 12, 18)
  context.fillStyle = '#e879f9'
  context.fillRect(6, player.value.height - 16, 10, 10)
  context.fillRect(player.value.width - 16, player.value.height - 16, 10, 10)

  context.restore()
}

const drawBullets = (context: CanvasRenderingContext2D) => {
  bullets.value.forEach((bullet) => {
    const gradient = context.createLinearGradient(bullet.x, bullet.y, bullet.x, bullet.y + bullet.height)
    gradient.addColorStop(0, '#ffffff')
    gradient.addColorStop(1, '#22d3ee')
    context.fillStyle = gradient
    context.fillRect(bullet.x, bullet.y, bullet.width, bullet.height)
  })
}

const drawEnemies = (context: CanvasRenderingContext2D) => {
  enemies.value.forEach((enemy) => {
    context.save()
    context.translate(enemy.x, enemy.y)

    const enemyGradient = context.createLinearGradient(0, 0, 0, enemy.height)
    if (enemy.elite) {
      enemyGradient.addColorStop(0, '#fdf2f8')
      enemyGradient.addColorStop(0.5, '#f472b6')
      enemyGradient.addColorStop(1, '#be185d')
    } else {
      enemyGradient.addColorStop(0, '#fef3c7')
      enemyGradient.addColorStop(0.5, '#f97316')
      enemyGradient.addColorStop(1, '#7c2d12')
    }

    context.fillStyle = enemyGradient
    context.beginPath()
    context.moveTo(enemy.width / 2, 0)
    context.lineTo(enemy.width, enemy.height * 0.28)
    context.lineTo(enemy.width - 6, enemy.height)
    context.lineTo(enemy.width / 2, enemy.height * 0.72)
    context.lineTo(6, enemy.height)
    context.lineTo(0, enemy.height * 0.28)
    context.closePath()
    context.fill()

    if (enemy.elite) {
      context.fillStyle = '#fdf2f8'
      context.fillRect(enemy.width / 2 - 10, 14, 20, 12)
      const hpRatio = enemy.hp / enemy.maxHp
      context.fillStyle = '#ffffff33'
      context.fillRect(6, enemy.height + 6, enemy.width - 12, 4)
      context.fillStyle = '#f472b6'
      context.fillRect(6, enemy.height + 6, (enemy.width - 12) * hpRatio, 4)
    } else {
      context.fillStyle = '#fde68a'
      context.fillRect(enemy.width / 2 - 6, 12, 12, 8)
    }

    context.restore()
  })
}

const drawParticles = (context: CanvasRenderingContext2D) => {
  particles.value.forEach((particle) => {
    context.save()
    context.globalAlpha = particle.life / particle.maxLife
    context.fillStyle = particle.color
    context.beginPath()
    context.arc(particle.x, particle.y, particle.size, 0, Math.PI * 2)
    context.fill()
    context.restore()
  })
}

const drawHud = (context: CanvasRenderingContext2D) => {
  context.save()
  context.fillStyle = '#020617cc'
  context.fillRect(14, 12, baseWidth - 28, 60)
  context.strokeStyle = '#22d3ee33'
  context.strokeRect(14, 12, baseWidth - 28, 60)

  context.fillStyle = '#94a3b8'
  context.font = '11px sans-serif'
  context.fillText('MISSION SCORE', 28, 30)
  context.fillText('SECTOR', 190, 30)
  context.fillText('HULL', baseWidth - 92, 30)

  context.fillStyle = '#e2e8f0'
  context.font = 'bold 20px sans-serif'
  context.fillText(String(score.value).padStart(4, '0'), 28, 54)
  context.fillStyle = '#67e8f9'
  context.fillText(String(level.value).padStart(2, '0'), 190, 54)
  context.fillStyle = player.value.lives <= 1 ? '#fb7185' : '#4ade80'
  context.fillText(String(Math.max(player.value.lives, 0)), baseWidth - 64, 54)

  context.fillStyle = '#94a3b8'
  context.font = '12px sans-serif'
  context.fillText('ARROW KEYS / SPACE', 20, baseHeight - 30)
  context.fillStyle = '#22d3ee'
  context.fillText('TACTICAL HUD ONLINE', baseWidth - 160, baseHeight - 30)
  context.restore()
}

const drawMobileControls = (context: CanvasRenderingContext2D) => {
  const joystickRadius = 42
  const knobRadius = 18
  const joystickKnobX = joystickCenter.value.x + joystickVector.value.x * 22
  const joystickKnobY = joystickCenter.value.y + joystickVector.value.y * 22

  context.save()
  context.globalAlpha = 0.92

  context.strokeStyle = '#67e8f9'
  context.lineWidth = 2
  context.fillStyle = '#08101fb8'
  context.beginPath()
  context.arc(joystickCenter.value.x, joystickCenter.value.y, joystickRadius, 0, Math.PI * 2)
  context.fill()
  context.stroke()

  context.beginPath()
  context.arc(joystickCenter.value.x, joystickCenter.value.y, joystickRadius - 12, 0, Math.PI * 2)
  context.strokeStyle = '#22d3ee22'
  context.stroke()

  context.fillStyle = '#22d3ee'
  context.beginPath()
  context.arc(joystickKnobX, joystickKnobY, knobRadius, 0, Math.PI * 2)
  context.fill()

  context.fillStyle = mobileFiring.value ? '#f472b6' : '#1e293bcc'
  context.strokeStyle = '#f9a8d4'
  context.beginPath()
  context.arc(fireButtonCenter.value.x, fireButtonCenter.value.y, 34, 0, Math.PI * 2)
  context.fill()
  context.stroke()

  context.beginPath()
  context.arc(fireButtonCenter.value.x, fireButtonCenter.value.y, 24, 0, Math.PI * 2)
  context.strokeStyle = '#f9a8d455'
  context.stroke()

  context.fillStyle = '#ffffff'
  context.font = 'bold 15px sans-serif'
  context.textAlign = 'center'
  context.fillText('FIRE', fireButtonCenter.value.x, fireButtonCenter.value.y + 5)
  context.restore()
}

const render = () => {
  const canvas = canvasRef.value
  if (!canvas) {
    return
  }

  const context = canvas.getContext('2d')
  if (!context) {
    return
  }

  context.save()
  context.clearRect(0, 0, canvas.width, canvas.height)
  const scaleX = canvas.clientWidth / baseWidth
  const scaleY = canvas.clientHeight / baseHeight
  context.scale(scaleX, scaleY)

  drawBackground(context)
  drawStars(context, farStars.value, '#dbeafe')
  drawStars(context, nearStars.value, '#67e8f9')
  drawBullets(context)
  drawEnemies(context)
  drawPlayer(context)
  drawParticles(context)
  drawHud(context)
  drawMobileControls(context)
  context.restore()
}

const tick = (timestamp: number) => {
  if (!lastTimestamp.value) {
    lastTimestamp.value = timestamp
  }

  const delta = Math.min((timestamp - lastTimestamp.value) / 1000, 0.024)
  lastTimestamp.value = timestamp

  updateStars(delta)

  if (props.phase === 'running' && player.value.lives > 0) {
    updatePlayer(delta)
    fireIfNeeded(delta)
    updateBullets(delta)
    updateEnemies(delta)
    handleCollisions()
    updateParticles(delta)
  } else {
    if (player.value.cooldown > 0) {
      player.value.cooldown = Math.max(0, player.value.cooldown - delta)
    }
    updateParticles(delta)
  }

  render()
  animationFrame.value = window.requestAnimationFrame(tick)
}

const updateJoystick = (clientX: number, clientY: number) => {
  const canvas = canvasRef.value
  if (!canvas) {
    return
  }

  const rect = canvas.getBoundingClientRect()
  const scaleX = baseWidth / rect.width
  const scaleY = baseHeight / rect.height
  const x = (clientX - rect.left) * scaleX
  const y = (clientY - rect.top) * scaleY
  const dx = x - joystickCenter.value.x
  const dy = y - joystickCenter.value.y
  const maxDistance = 26
  const distance = Math.hypot(dx, dy)
  const ratio = distance > maxDistance ? maxDistance / distance : 1

  joystickVector.value = {
    x: clamp((dx * ratio) / maxDistance, -1, 1),
    y: clamp((dy * ratio) / maxDistance, -1, 1),
  }
}

const isInsideCircle = (clientX: number, clientY: number, centerX: number, centerY: number, radius: number) => {
  const canvas = canvasRef.value
  if (!canvas) {
    return false
  }

  const rect = canvas.getBoundingClientRect()
  const scaleX = baseWidth / rect.width
  const scaleY = baseHeight / rect.height
  const x = (clientX - rect.left) * scaleX
  const y = (clientY - rect.top) * scaleY
  return Math.hypot(x - centerX, y - centerY) <= radius
}

const handlePointerDown = (event: PointerEvent) => {
  const target = event.target
  if (!(target instanceof HTMLCanvasElement)) {
    return
  }

  if (props.phase !== 'running') {
    return
  }

  if (isInsideCircle(event.clientX, event.clientY, joystickCenter.value.x, joystickCenter.value.y, 54)) {
    joystickActive.value = true
    joystickPointerId.value = event.pointerId
    updateJoystick(event.clientX, event.clientY)
    target.setPointerCapture(event.pointerId)
    return
  }

  if (isInsideCircle(event.clientX, event.clientY, fireButtonCenter.value.x, fireButtonCenter.value.y, 40)) {
    firePointerId.value = event.pointerId
    mobileFiring.value = true
    target.setPointerCapture(event.pointerId)
  }
}

const handlePointerMove = (event: PointerEvent) => {
  if (!joystickActive.value || joystickPointerId.value !== event.pointerId) {
    return
  }

  updateJoystick(event.clientX, event.clientY)
}

const clearJoystick = () => {
  joystickActive.value = false
  joystickPointerId.value = null
  joystickVector.value = { x: 0, y: 0 }
}

const clearFiring = () => {
  firePointerId.value = null
  mobileFiring.value = false
}

const handlePointerUp = (event: PointerEvent) => {
  if (joystickPointerId.value === event.pointerId) {
    clearJoystick()
  }

  if (firePointerId.value === event.pointerId) {
    clearFiring()
  }
}

const onKeyDown = (event: KeyboardEvent) => {
  if (['ArrowLeft', 'ArrowRight', 'ArrowUp', 'ArrowDown', 'Space'].includes(event.code)) {
    event.preventDefault()
    keys.value[event.code] = true
  }
}

const onKeyUp = (event: KeyboardEvent) => {
  if (['ArrowLeft', 'ArrowRight', 'ArrowUp', 'ArrowDown', 'Space'].includes(event.code)) {
    event.preventDefault()
    keys.value[event.code] = false
  }
}

watch(
  () => props.restartToken,
  () => {
    resetGameState()
  },
)

watch(
  () => props.phase,
  (phase) => {
    if (phase !== 'running') {
      mobileFiring.value = false
      clearJoystick()
    }
  },
)

onMounted(() => {
  initStars()
  resizeCanvas()
  window.addEventListener('resize', resizeCanvas)
  window.addEventListener('keydown', onKeyDown)
  window.addEventListener('keyup', onKeyUp)
  animationFrame.value = window.requestAnimationFrame(tick)
})

onBeforeUnmount(() => {
  if (animationFrame.value !== null) {
    window.cancelAnimationFrame(animationFrame.value)
  }
  window.removeEventListener('resize', resizeCanvas)
  window.removeEventListener('keydown', onKeyDown)
  window.removeEventListener('keyup', onKeyUp)
})
</script>

<template>
  <div ref="wrapperRef" class="relative h-full w-full">
    <canvas
      ref="canvasRef"
      class="h-full w-full touch-none"
      @pointerdown="handlePointerDown"
      @pointermove="handlePointerMove"
      @pointerup="handlePointerUp"
      @pointercancel="handlePointerUp"
      @pointerleave="handlePointerUp"
    />
  </div>
</template>
