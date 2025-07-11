<script setup>
import { ref, onMounted, onUnmounted, reactive } from 'vue'

const canvas = ref(null)
let ctx = null
let confetti = []
let trail = []
let stars = []
let animationId = null
let mouseX = 0
let mouseY = 0
let isMouseInside = false
let isPainting = false
let shakeX = ref(0)
let shakeY = ref(0)
let shakeIntensity = 0
let shakeDecay = 0.9

// Dropdown state
const isDropdownOpen = ref(false)

// Feature toggles
const features = reactive({
  starfield: true,
  screenShake: true,
  particlePainting: true,
  trailParticles: true,
  lightEffect: true,
  confetti: true
})

const colors = ['#ff6b6b', '#4ecdc4', '#45b7d1', '#f9ca24', '#f0932b', '#eb4d4b', '#6c5ce7', '#a29bfe', '#fd79a8', '#e84393']
const trailColors = ['#ffffff', '#ffd700', '#ffb6c1', '#87ceeb', '#98fb98', '#dda0dd', '#f0e68c', '#ff69b4']
const paintColors = ['#ff0080', '#00ff80', '#8000ff', '#ff8000', '#0080ff', '#80ff00', '#ff0040', '#40ff00']

class Star {
  constructor() {
    this.x = Math.random() * window.innerWidth
    this.y = Math.random() * window.innerHeight
    this.size = Math.random() * 2 + 0.5
    this.opacity = Math.random() * 0.8 + 0.2
    this.twinkleSpeed = Math.random() * 0.02 + 0.01
    this.twinkleDirection = Math.random() > 0.5 ? 1 : -1
    this.maxOpacity = this.opacity
    this.minOpacity = this.opacity * 0.3
  }

  update() {
    this.opacity += this.twinkleSpeed * this.twinkleDirection
    if (this.opacity >= this.maxOpacity) {
      this.opacity = this.maxOpacity
      this.twinkleDirection = -1
    } else if (this.opacity <= this.minOpacity) {
      this.opacity = this.minOpacity
      this.twinkleDirection = 1
    }
  }

  draw() {
    ctx.save()
    ctx.globalAlpha = this.opacity
    ctx.beginPath()
    ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2)
    ctx.fillStyle = '#ffffff'
    ctx.fill()
    
    // Add star twinkle effect
    ctx.shadowColor = '#ffffff'
    ctx.shadowBlur = 5
    ctx.beginPath()
    ctx.arc(this.x, this.y, this.size * 0.5, 0, Math.PI * 2)
    ctx.fillStyle = '#ffffff'
    ctx.fill()
    
    ctx.restore()
  }
}

class ConfettiPiece {
  constructor(x, y) {
    this.x = x
    this.y = y
    this.vx = (Math.random() - 0.5) * 10
    this.vy = Math.random() * -15 - 5
    this.gravity = 0.5
    this.friction = 0.99
    this.color = colors[Math.floor(Math.random() * colors.length)]
    this.size = Math.random() * 8 + 4
    this.rotation = Math.random() * 360
    this.rotationSpeed = (Math.random() - 0.5) * 15
    this.opacity = 1
    this.decay = Math.random() * 0.02 + 0.005
  }

  update() {
    this.vx *= this.friction
    this.vy += this.gravity
    this.x += this.vx
    this.y += this.vy
    this.rotation += this.rotationSpeed
    this.opacity -= this.decay
  }

  draw() {
    ctx.save()
    ctx.globalAlpha = this.opacity
    ctx.translate(this.x, this.y)
    ctx.rotate(this.rotation * Math.PI / 180)
    ctx.fillStyle = this.color
    ctx.fillRect(-this.size / 2, -this.size / 2, this.size, this.size)
    ctx.restore()
  }
}

class TrailParticle {
  constructor(x, y, isPaintMode = false) {
    this.x = x + (Math.random() - 0.5) * 10
    this.y = y + (Math.random() - 0.5) * 10
    this.vx = (Math.random() - 0.5) * 2
    this.vy = (Math.random() - 0.5) * 2
    this.color = isPaintMode ? 
      paintColors[Math.floor(Math.random() * paintColors.length)] : 
      trailColors[Math.floor(Math.random() * trailColors.length)]
    this.size = isPaintMode ? Math.random() * 6 + 3 : Math.random() * 4 + 2
    this.opacity = 1
    this.decay = Math.random() * 0.02 + 0.01
    this.life = 1
    this.isPaintMode = isPaintMode
  }

  update() {
    this.x += this.vx
    this.y += this.vy
    this.vx *= 0.98
    this.vy *= 0.98
    this.opacity -= this.decay
    this.life -= 0.02
    this.size *= 0.99
  }

  draw() {
    ctx.save()
    ctx.globalAlpha = this.opacity
    ctx.beginPath()
    ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2)
    ctx.fillStyle = this.color
    ctx.fill()
    
    // Add a glowing effect
    ctx.shadowColor = this.color
    ctx.shadowBlur = this.isPaintMode ? 15 : 10
    ctx.beginPath()
    ctx.arc(this.x, this.y, this.size * 0.5, 0, Math.PI * 2)
    ctx.fillStyle = this.color
    ctx.fill()
    
    ctx.restore()
  }
}

// Feature toggle functions
const toggleDropdown = () => {
  isDropdownOpen.value = !isDropdownOpen.value
}

const toggleStarfield = () => {
  if (!features.starfield) {
    stars = []
  } else {
    createStars()
  }
}

const toggleScreenShake = () => {
  if (!features.screenShake) {
    shakeIntensity = 0
    shakeX.value = 0
    shakeY.value = 0
  }
}

const toggleParticlePainting = () => {
  if (!features.particlePainting) {
    isPainting = false
  }
}

const toggleTrailParticles = () => {
  if (!features.trailParticles) {
    trail = []
  }
}

const toggleLightEffect = () => {
  // Light effect is handled in the draw function
}

const toggleConfetti = () => {
  if (!features.confetti) {
    confetti = []
  }
}

const createStars = () => {
  stars = []
  const numStars = 100
  for (let i = 0; i < numStars; i++) {
    stars.push(new Star())
  }
}

const updateScreenShake = () => {
  if (shakeIntensity > 0 && features.screenShake) {
    shakeX.value = (Math.random() - 0.5) * shakeIntensity
    shakeY.value = (Math.random() - 0.5) * shakeIntensity
    shakeIntensity *= shakeDecay
    
    if (shakeIntensity < 0.1) {
      shakeIntensity = 0
      shakeX.value = 0
      shakeY.value = 0
    }
  }
}

const triggerScreenShake = () => {
  if (features.screenShake) {
    shakeIntensity = 10
  }
}

const resizeCanvas = () => {
  if (canvas.value) {
    canvas.value.width = window.innerWidth
    canvas.value.height = window.innerHeight
    if (features.starfield) {
      createStars() // Recreate stars on resize
    }
  }
}

const drawLightEffect = () => {
  if (!ctx || !isMouseInside || !features.lightEffect) return
  
  // Create radial gradient for light effect
  const gradient = ctx.createRadialGradient(mouseX, mouseY, 0, mouseX, mouseY, 200)
  gradient.addColorStop(0, 'rgba(255, 255, 255, 0.3)')
  gradient.addColorStop(0.3, 'rgba(255, 255, 255, 0.1)')
  gradient.addColorStop(0.7, 'rgba(255, 255, 255, 0.05)')
  gradient.addColorStop(1, 'rgba(255, 255, 255, 0)')
  
  ctx.save()
  ctx.globalCompositeOperation = 'screen'
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, canvas.value.width, canvas.value.height)
  ctx.restore()
}

const throwConfetti = (event) => {
  if (!features.confetti) return
  
  const rect = canvas.value.getBoundingClientRect()
  const x = event.clientX - rect.left
  const y = event.clientY - rect.top
  
  // Create 50 confetti pieces at click location
  for (let i = 0; i < 50; i++) {
    confetti.push(new ConfettiPiece(x, y))
  }
  
  // Trigger screen shake
  triggerScreenShake()
}

const startPainting = () => {
  if (features.particlePainting) {
    isPainting = true
  }
}

const stopPainting = () => {
  isPainting = false
}

const onMouseMove = (event) => {
  const rect = canvas.value.getBoundingClientRect()
  mouseX = event.clientX - rect.left
  mouseY = event.clientY - rect.top
  isMouseInside = true
  
  if (!features.trailParticles) return
  
  // Create trail particles
  const particleChance = (isPainting && features.particlePainting) ? 0.9 : 0.7
  if (Math.random() < particleChance) {
    trail.push(new TrailParticle(mouseX, mouseY, isPainting && features.particlePainting))
  }
  
  // Create extra particles when painting
  if (isPainting && features.particlePainting && Math.random() < 0.8) {
    trail.push(new TrailParticle(mouseX, mouseY, true))
  }
}

const onMouseLeave = () => {
  isMouseInside = false
  stopPainting()
}

const animate = () => {
  if (!ctx) return
  
  // Fill with black background
  ctx.fillStyle = '#000000'
  ctx.fillRect(0, 0, canvas.value.width, canvas.value.height)
  
  // Update and draw stars
  if (features.starfield) {
    stars.forEach(star => {
      star.update()
      star.draw()
    })
  }
  
  // Draw light effect
  drawLightEffect()
  
  // Update and draw trail particles
  if (features.trailParticles) {
    trail = trail.filter(particle => {
      particle.update()
      particle.draw()
      return particle.opacity > 0 && particle.life > 0
    })
  }
  
  // Update and draw confetti
  if (features.confetti) {
    confetti = confetti.filter(piece => {
      piece.update()
      piece.draw()
      return piece.opacity > 0 && piece.y < canvas.value.height + 100
    })
  }
  
  // Update screen shake
  updateScreenShake()
  
  animationId = requestAnimationFrame(animate)
}

onMounted(() => {
  if (canvas.value) {
    ctx = canvas.value.getContext('2d')
    resizeCanvas()
    createStars()
    animate()
  }
  
  window.addEventListener('resize', resizeCanvas)
  document.addEventListener('mouseleave', onMouseLeave)
})

onUnmounted(() => {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }
  window.removeEventListener('resize', resizeCanvas)
  document.removeEventListener('mouseleave', onMouseLeave)
})
</script>

<template>
  <div class="confetti-container" 
       @click="throwConfetti" 
       @mousemove="onMouseMove"
       @mousedown="startPainting"
       @mouseup="stopPainting"
       @mouseleave="stopPainting"
       :style="{ transform: `translate(${shakeX}px, ${shakeY}px)` }">
    <canvas ref="canvas" class="confetti-canvas"></canvas>
    
    <!-- Features Dropdown -->
    <div class="features-dropdown" :class="{ 'expanded': isDropdownOpen }">
      <button class="dropdown-toggle" @click="toggleDropdown">
        <span>✨ Features</span>
        <span class="arrow" :class="{ 'rotated': isDropdownOpen }">▼</span>
      </button>
      
      <div class="dropdown-content" v-show="isDropdownOpen">
        <div class="feature-item">
          <label class="feature-label">
            <input type="checkbox" v-model="features.starfield" @change="toggleStarfield">
            <span class="checkmark"></span>
            🌌 Starfield Background
          </label>
        </div>
        
        <div class="feature-item">
          <label class="feature-label">
            <input type="checkbox" v-model="features.screenShake" @change="toggleScreenShake">
            <span class="checkmark"></span>
            💥 Screen Shake
          </label>
        </div>
        
        <div class="feature-item">
          <label class="feature-label">
            <input type="checkbox" v-model="features.particlePainting" @change="toggleParticlePainting">
            <span class="checkmark"></span>
            🎨 Particle Painting
          </label>
        </div>
        
        <div class="feature-item">
          <label class="feature-label">
            <input type="checkbox" v-model="features.trailParticles" @change="toggleTrailParticles">
            <span class="checkmark"></span>
            ✨ Mouse Trail
          </label>
        </div>
        
        <div class="feature-item">
          <label class="feature-label">
            <input type="checkbox" v-model="features.lightEffect" @change="toggleLightEffect">
            <span class="checkmark"></span>
            💡 Cursor Light
          </label>
        </div>
        
        <div class="feature-item">
          <label class="feature-label">
            <input type="checkbox" v-model="features.confetti" @change="toggleConfetti">
            <span class="checkmark"></span>
            🎉 Confetti
          </label>
        </div>
      </div>
    </div>
    
    <div class="content">
      <h1 class="title">🎉 Confetti! 🎉</h1>
      <p class="instructions" v-if="features.trailParticles">Move your mouse to see the sparkly trail ✨</p>
      <p class="instructions" v-if="features.confetti">Click anywhere to throw confetti <span v-if="features.screenShake">and shake the screen</span> 🎉</p>
      <p class="instructions" v-if="features.lightEffect">Move your mouse to see the light effect 💡</p>
      <p class="instructions" v-if="features.particlePainting">Hold mouse down to paint with particles 🎨</p>
    </div>
  </div>
</template>

<style scoped>
.confetti-container {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background: #000000;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.confetti-canvas {
  position: absolute;
  top: 0;
  left: 0;
  pointer-events: none;
  z-index: 1;
}

.features-dropdown {
  position: absolute;
  top: 20px;
  right: 20px;
  z-index: 10;
  font-family: Arial, sans-serif;
}

.dropdown-toggle {
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.3);
  color: white;
  padding: 12px 16px;
  border-radius: 8px;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 16px;
  backdrop-filter: blur(10px);
  transition: all 0.3s ease;
}

.dropdown-toggle:hover {
  background: rgba(255, 255, 255, 0.2);
  border-color: rgba(255, 255, 255, 0.5);
}

.arrow {
  transition: transform 0.3s ease;
}

.arrow.rotated {
  transform: rotate(180deg);
}

.dropdown-content {
  position: absolute;
  top: 100%;
  right: 0;
  background: rgba(0, 0, 0, 0.8);
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 8px;
  backdrop-filter: blur(10px);
  padding: 12px;
  min-width: 220px;
  margin-top: 8px;
  animation: slideDown 0.3s ease;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.feature-item {
  margin-bottom: 8px;
}

.feature-item:last-child {
  margin-bottom: 0;
}

.feature-label {
  display: flex;
  align-items: center;
  color: white;
  cursor: pointer;
  padding: 8px;
  border-radius: 4px;
  transition: background-color 0.3s ease;
  font-size: 14px;
}

.feature-label:hover {
  background: rgba(255, 255, 255, 0.1);
}

.feature-label input[type="checkbox"] {
  opacity: 0;
  position: absolute;
  width: 0;
  height: 0;
}

.checkmark {
  width: 18px;
  height: 18px;
  background: rgba(255, 255, 255, 0.2);
  border: 2px solid rgba(255, 255, 255, 0.5);
  border-radius: 3px;
  margin-right: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
}

.feature-label input[type="checkbox"]:checked + .checkmark {
  background: #4CAF50;
  border-color: #4CAF50;
}

.feature-label input[type="checkbox"]:checked + .checkmark::after {
  content: "✓";
  color: white;
  font-weight: bold;
  font-size: 12px;
}

.content {
  text-align: center;
  z-index: 2;
  color: white;
  user-select: none;
}

.title {
  font-size: 3rem;
  font-weight: bold;
  margin-bottom: 1rem;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.8);
  animation: pulse 2s ease-in-out infinite alternate;
}

.instructions {
  font-size: 1.2rem;
  opacity: 0.8;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.8);
  font-style: italic;
  margin-bottom: 0.5rem;
}

@keyframes pulse {
  from {
    transform: scale(1);
  }
  to {
    transform: scale(1.05);
  }
}

@media (max-width: 768px) {
  .features-dropdown {
    top: 10px;
    right: 10px;
  }
  
  .dropdown-toggle {
    padding: 10px 12px;
    font-size: 14px;
  }
  
  .dropdown-content {
    min-width: 180px;
  }
  
  .title {
    font-size: 2rem;
  }
  
  .instructions {
    font-size: 1rem;
  }
}
</style>
