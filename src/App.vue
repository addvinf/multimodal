<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'

interface Player {
  x: number;
  y: number;
  speed: number;
  radius: number;
  color: string;
}

const canvasWidth = 500
const canvasHeight = 500

// Reactive player states
const player1 = ref<Player>({ x: 100, y: 100, speed: 3, radius: 15, color: 'blue' })
const player2 = ref<Player>({ x: 400, y: 400, speed: 3, radius: 15, color: 'red' })

const keysPressed = new Set<string>()

// Game state
const gameOver = ref(false)
const gameStarted = ref(false)

// Timer
const time = ref(0)
let timeInterval: number | undefined

// Audio
let audioContext: AudioContext
let oscillator: OscillatorNode
let gainNode: GainNode

// Hide/Show game field
const hideGamePlan = ref(false) // Controls whether the field is hidden or shown

const initializeAudio = () => {
  if (!audioContext) {
    audioContext = new AudioContext()
    oscillator = audioContext.createOscillator()
    gainNode = audioContext.createGain()
    oscillator.connect(gainNode).connect(audioContext.destination)
    oscillator.frequency.value = 440
    oscillator.start()
  }
}

const updateSoundVolume = () => {
  if (!gainNode) return
  const dx = player1.value.x - player2.value.x
  const dy = player1.value.y - player2.value.y
  const distance = Math.sqrt(dx * dx + dy * dy)

  const maxDistance = Math.sqrt(canvasWidth * canvasWidth + canvasHeight * canvasHeight)
  
  const cutoffDistance = maxDistance * 0.7
  let volume = 0
  if (distance < cutoffDistance) {
    const normalized = 1 - (distance / cutoffDistance)
    volume = 0.5 * (normalized ** 2)
  } else {
    volume = 0
  }

  gainNode.gain.value = volume
}

// Check collision
const checkCollision = () => {
  const dx = player1.value.x - player2.value.x
  const dy = player1.value.y - player2.value.y
  const distance = Math.sqrt(dx * dx + dy * dy)
  return distance <= (player1.value.radius + player2.value.radius)
}

const movePlayers = () => {
  if (gameOver.value) return

  const oldX1 = player1.value.x
  const oldY1 = player1.value.y
  const oldX2 = player2.value.x
  const oldY2 = player2.value.y

  // Player 1 - WASD
  if (keysPressed.has('w')) player1.value.y -= player1.value.speed
  if (keysPressed.has('s')) player1.value.y += player1.value.speed
  if (keysPressed.has('a')) player1.value.x -= player1.value.speed
  if (keysPressed.has('d')) player1.value.x += player1.value.speed

  // Player 2 - Arrow Keys
  if (keysPressed.has('ArrowUp')) player2.value.y -= player2.value.speed
  if (keysPressed.has('ArrowDown')) player2.value.y += player2.value.speed
  if (keysPressed.has('ArrowLeft')) player2.value.x -= player2.value.speed
  if (keysPressed.has('ArrowRight')) player2.value.x += player2.value.speed

  // Constrain players inside field
  player1.value.x = Math.max(player1.value.radius, Math.min(canvasWidth - player1.value.radius, player1.value.x))
  player1.value.y = Math.max(player1.value.radius, Math.min(canvasHeight - player1.value.radius, player1.value.y))
  
  player2.value.x = Math.max(player2.value.radius, Math.min(canvasWidth - player2.value.radius, player2.value.x))
  player2.value.y = Math.max(player2.value.radius, Math.min(canvasHeight - player2.value.radius, player2.value.y))

  // Start the game & timer on the first valid move
  if (!gameStarted.value && (
    player1.value.x !== oldX1 || player1.value.y !== oldY1 ||
    player2.value.x !== oldX2 || player2.value.y !== oldY2
  )) {
    startGameTimer()
    gameStarted.value = true
  }
}

const startGameTimer = () => {
  time.value = 0
  clearInterval(timeInterval)
  timeInterval = window.setInterval(() => {
    time.value++
  }, 1000)
}

const stopGameTimer = () => {
  if (timeInterval) {
    clearInterval(timeInterval)
    timeInterval = undefined
  }
}

// Drawing
let ctx: CanvasRenderingContext2D | null = null
const gameCanvas = ref<HTMLCanvasElement | null>(null)

const draw = () => {
  if (!ctx) return

  // If hideGamePlan is true, do not visually draw anything.
  // But the canvas is still there with v-show.
  if (hideGamePlan.value) {
    ctx.clearRect(0, 0, canvasWidth, canvasHeight)
    return
  }

  ctx.clearRect(0, 0, canvasWidth, canvasHeight)
  ctx.strokeStyle = '#333'
  ctx.strokeRect(0, 0, canvasWidth, canvasHeight)

  // Player 1
  ctx.fillStyle = player1.value.color
  ctx.beginPath()
  ctx.arc(player1.value.x, player1.value.y, player1.value.radius, 0, Math.PI * 2)
  ctx.fill()

  // Player 2
  ctx.fillStyle = player2.value.color
  ctx.beginPath()
  ctx.arc(player2.value.x, player2.value.y, player2.value.radius, 0, Math.PI * 2)
  ctx.fill()
}

let animationFrameId: number
const gameLoop = () => {
  movePlayers()
  updateSoundVolume()
  draw()

  // Check for collision and end game if needed
  if (checkCollision()) {
    endGame()
  } else {
    animationFrameId = requestAnimationFrame(gameLoop)
  }
}

const endGame = () => {
  gameOver.value = true
  stopGameTimer()
  cancelAnimationFrame(animationFrameId)

  // Mute the audio
  if (gainNode) {
    gainNode.gain.value = 0
  }
}

const restartGame = () => {
  // Reset players
  player1.value = { x: 100, y: 100, speed: 3, radius: 15, color: 'blue' }
  player2.value = { x: 400, y: 400, speed: 3, radius: 15, color: 'red' }

  // Reset states
  keysPressed.clear()
  gameOver.value = false
  gameStarted.value = false
  time.value = 0

  // Resume animation loop
  animationFrameId = requestAnimationFrame(gameLoop)
}

// Function to toggle the visibility of the game plan
const toggleGamePlan = () => {
  hideGamePlan.value = !hideGamePlan.value
}

onMounted(() => {
  const canvas = gameCanvas.value
  if (!canvas) return
  ctx = canvas.getContext('2d') as CanvasRenderingContext2D

  const handleKeyDown = (e: KeyboardEvent) => {
    keysPressed.add(e.key)
    // Initialize audio on first key press to comply with browser audio policies
    if (!audioContext) {
      initializeAudio()
    } else if (audioContext.state === 'suspended') {
      audioContext.resume()
    }
  }

  const handleKeyUp = (e: KeyboardEvent) => {
    keysPressed.delete(e.key)
  }

  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('keyup', handleKeyUp)

  // Start the loop
  animationFrameId = requestAnimationFrame(gameLoop)
})

onBeforeUnmount(() => {
  cancelAnimationFrame(animationFrameId)
  window.removeEventListener('keydown', () => {})
  window.removeEventListener('keyup', () => {})
  if (oscillator) {
    oscillator.stop()
    oscillator.disconnect()
  }
  if (audioContext) {
    audioContext.close()
  }
  stopGameTimer()
})
</script>

<template>
  <div style="position: relative; width: 500px; margin: 0 auto;">
    <div style="display:flex; justify-content:space-between; margin-bottom:10px;">
      <button @click="toggleGamePlan">{{ hideGamePlan ? 'Show' : 'Hide' }} Game Plan</button>
    </div>

    <!-- Use v-show instead of v-if so the canvas remains in the DOM -->
    <canvas v-show="!hideGamePlan" ref="gameCanvas" :width="500" :height="500" style="border:1px solid #000;"></canvas>

    <div v-if="gameOver" style="position: absolute; top: 220px; left: 180px; background: white; padding: 10px; border: 1px solid #000;">
      <p>Game Over!</p>
      <button @click="restartGame">Restart</button>
    </div>
    <!-- Timer display on the right (only visible if game is not over and the plan is not hidden) -->
    <div v-if="!gameOver && !hideGamePlan" style="position: absolute; top: 10px; right: 10px; background: white; padding: 5px; border: 1px solid #000;">
      <p>Time: {{ time }}s</p>
    </div>
  </div>
</template>

<style scoped>
canvas {
  display: block;
  margin-top: 50px;
  background: #eee;
}
</style>