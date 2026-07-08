<template>
  <div class="game-container">
    <!-- Header Controls -->
    <header class="game-header">
      <div class="mode-selector">
        <button 
          :class="{ active: interactionMode === 'raise' }" 
          @click="interactionMode = 'raise'"
        >
          ▲ Raise
        </button>
        <button 
          :class="{ active: interactionMode === 'lower' }" 
          @click="interactionMode = 'lower'"
        >
          ▼ Lower
        </button>
        <button 
          :class="{ active: interactionMode === 'rotate' }" 
          @click="interactionMode = 'rotate'"
        >
          ↻ Rotate
        </button>
      </div>
    </header>

    <!-- Canvas Viewport Container -->
    <div class="viewport-wrapper" ref="wrapper">
      <canvas 
        ref="landscapeCanvas"
        @touchstart.passive="handleTouchStart"
        @touchmove.passive="handleTouchMove"
        @touchend="handleTouchEnd"
        @mousedown="handleMouseDown"
        @mousemove="handleMouseMove"
        @mouseup="handleMouseUp"
      ></canvas>
    </div>

    <!-- Mobile Footer Info -->
    <footer class="game-footer">
      <p v-if="interactionMode === 'rotate'">Drag to rotate view</p>
      <p v-else>Tap land to alter the biomes</p>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue'

// Canvas & Layout Refs
const landscapeCanvas = ref(null)
const wrapper = ref(null)
let ctx = null
let animationFrameId = null

// Game/Engine State
const GRID_SIZE = 14 // 14x14 grid
const interactionMode = ref('raise') 
const angle = ref(0.75) // Camera rotation angle

// Input Tracking
const isInteracting = ref(false)
let lastInputX = 0
let lastInputY = 0

// SEA LEVEL DEFINITION (Anything below this is water)
const SEA_LEVEL = 5

// Generate the initial landscape height map
const heightMap = ref([])
for (let x = 0; x <= GRID_SIZE; x++) {
  heightMap.value[x] = []
  for (let y = 0; y <= GRID_SIZE; y++) {
    // Generate a central landmass island to start
    const distanceFromCenter = Math.hypot(x - GRID_SIZE/2, y - GRID_SIZE/2)
    if (distanceFromCenter < 4) {
      heightMap.value[x][y] = 25 // High grass/mountain peak
    } else if (distanceFromCenter < 6) {
      heightMap.value[x][y] = 12 // Flat grass plain
    } else {
      heightMap.value[x][y] = 0 // Submerged water trench
    }
  }
}

// Isometric Projection Math
const project = (x, y, z, width, height) => {
  const cx = width / 2
  const cy = height / 1.9 // Slightly lower down center
  const scale = Math.min(width, height) / (GRID_SIZE * 1.6)
  
  const halfGrid = GRID_SIZE / 2
  const rx = (x - halfGrid) * Math.cos(angle.value) - (y - halfGrid) * Math.sin(angle.value)
  const ry = (x - halfGrid) * Math.sin(angle.value) + (y - halfGrid) * Math.cos(angle.value)
  
  const screenX = cx + (rx - ry) * scale
  // We clamp visual rendering of underwater coordinates to look like a flat ocean floor bed
  const renderedZ = Math.max(SEA_LEVEL, z)
  const screenY = cy + (rx + ry) * (scale * 0.5) - renderedZ
  
  return { x: screenX, y: screenY }
}

// Main Render Loop
const drawLandscape = () => {
  if (!landscapeCanvas.value || !ctx) return
  
  const canvas = landscapeCanvas.value
  const w = canvas.width / (window.devicePixelRatio || 1)
  const h = canvas.height / (window.devicePixelRatio || 1)
  
  // Clean background frame
  ctx.fillStyle = '#0b0f19'
  ctx.fillRect(0, 0, w, h)
  
  // Render grid boxes back-to-front painter algorithm
  for (let x = 0; x < GRID_SIZE; x++) {
    for (let y = 0; y < GRID_SIZE; y++) {
      // 1. Fetch Elevation values for the 4 corners of the quad mesh
      const h1 = heightMap.value[x][y]
      const h2 = heightMap.value[x + 1][y]
      const h3 = heightMap.value[x + 1][y + 1]
      const h4 = heightMap.value[x][y + 1]
      
      const avgHeight = (h1 + h2 + h3 + h4) / 4
      
      // Calculate projected screen pixel dots
      const p1 = project(x, y, h1, w, h)
      const p2 = project(x + 1, y, h2, w, h)
      const p3 = project(x + 1, y + 1, h3, w, h)
      const p4 = project(x, y + 1, h4, w, h)
      
      ctx.beginPath()
      ctx.moveTo(p1.x, p1.y)
      ctx.lineTo(p2.x, p2.y)
      ctx.lineTo(p3.x, p3.y)
      ctx.lineTo(p4.x, p4.y)
      ctx.closePath()
      
      // 2. BIOME COLOR LOGIC (Dynamic thematic mapping based on height)
      let fillStyle = ''
      let strokeStyle = ''
      
      if (avgHeight <= SEA_LEVEL) {
        // Deep Ocean
        fillStyle = '#112244' 
        strokeStyle = '#1d3557'
      } else if (avgHeight <= SEA_LEVEL + 4) {
        // Sand Beaches
        fillStyle = '#c5a059' 
        strokeStyle = '#d4af37'
      } else if (avgHeight <= SEA_LEVEL + 24) {
        // Fertile Grass Plain
        fillStyle = '#1e4620' 
        strokeStyle = '#2e6f40'
      } else if (avgHeight <= SEA_LEVEL + 45) {
        // High Mountain rock
        fillStyle = '#4a4e69' 
        strokeStyle = '#9a8c98'
      } else {
        // Snowy Peak summits
        fillStyle = '#f2e9e4' 
        strokeStyle = '#ffffff'
      }
      
      ctx.fillStyle = fillStyle
      ctx.fill()
      
      ctx.strokeStyle = strokeStyle
      ctx.lineWidth = 1
      ctx.stroke()
    }
  }
  
  animationFrameId = requestAnimationFrame(drawLandscape)
}

// Modify point height along with immediate neighbors for smoother sloped terrain hills
const alterTerrainSmoothly = (targetX, targetY, amount) => {
  for (let x = 0; x <= GRID_SIZE; x++) {
    for (let y = 0; y <= GRID_SIZE; y++) {
      const distance = Math.hypot(x - targetX, y - targetY)
      
      // Falloff modifier radius: 100% impact at tap point, drops off up to 2 units away
      if (distance < 2.5) {
        const falloff = 1 - (distance / 2.5)
        const addition = amount * falloff
        
        heightMap.value[x][y] = Math.max(0, Math.min(75, heightMap.value[x][y] + addition))
      }
    }
  }
}

// Input Core Controllers
const processInputMove = (currentX, currentY) => {
  if (!isInteracting.value) return
  const deltaX = currentX - lastInputX
  
  if (interactionMode.value === 'rotate') {
    angle.value += deltaX * 0.007
  }
  
  lastInputX = currentX
  lastInputY = currentY
}

const processInputDown = (clientX, clientY) => {
  if (!landscapeCanvas.value) return
  isInteracting.value = true
  
  const rect = landscapeCanvas.value.getBoundingClientRect()
  const canvasX = clientX - rect.left
  const canvasY = clientY - rect.top
  
  lastInputX = canvasX
  lastInputY = canvasY
  
  if (interactionMode.value === 'raise' || interactionMode.value === 'lower') {
    let closestX = -1
    let closestY = -1
    let minDistance = 35 
    
    const w = landscapeCanvas.value.width / (window.devicePixelRatio || 1)
    const h = landscapeCanvas.value.height / (window.devicePixelRatio || 1)
    
    for (let x = 0; x <= GRID_SIZE; x++) {
      for (let y = 0; y <= GRID_SIZE; y++) {
        // Calculate point relative positions
        const p = project(x, y, heightMap.value[x][y], w, h)
        const dist = Math.hypot(p.x - canvasX, p.y - canvasY)
        
        if (dist < minDistance) {
          minDistance = dist
          closestX = x
          closestY = y
        }
      }
    }
    
    if (closestX !== -1 && closestY !== -1) {
      const power = interactionMode.value === 'raise' ? 12 : -12
      alterTerrainSmoothly(closestX, closestY, power)
    }
  }
}

// Mobile Extraction Maps
const handleTouchStart = (e) => {
  if (e.touches && e.touches.length === 1) {
    processInputDown(e.touches[0].clientX, e.touches[0].clientY)
  }
}
const handleTouchMove = (e) => {
  if (e.touches && e.touches.length === 1 && landscapeCanvas.value) {
    const rect = landscapeCanvas.value.getBoundingClientRect()
    processInputMove(e.touches[0].clientX - rect.left, e.touches[0].clientY - rect.top)
  }
}
const handleTouchEnd = () => { isInteracting.value = false }

// Desktop Mouse Wrappers
const handleMouseDown = (e) => processInputDown(e.clientX, e.clientY)
const handleMouseMove = (e) => {
  if (landscapeCanvas.value) {
    const rect = landscapeCanvas.value.getBoundingClientRect()
    processInputMove(e.clientX - rect.left, e.clientY - rect.top)
  }
}
const handleMouseUp = () => { isInteracting.value = false }

// Dynamic Canvas Auto Sizing Adjuster
const resizeCanvas = () => {
  if (!landscapeCanvas.value || !wrapper.value) return
  const canvas = landscapeCanvas.value
  const dpr = window.devicePixelRatio || 1
  
  const targetWidth = wrapper.value.offsetWidth || window.innerWidth
  const targetHeight = wrapper.value.offsetHeight || (window.innerHeight - 140)
  
  canvas.width = targetWidth * dpr
  canvas.height = targetHeight * dpr
  canvas.style.width = `${targetWidth}px`
  canvas.style.height = `${targetHeight}px`
  
  ctx = canvas.getContext('2d')
  ctx.scale(dpr, dpr)
}

onMounted(async () => {
  await nextTick()
  setTimeout(() => {
    resizeCanvas()
    window.addEventListener('resize', resizeCanvas)
    drawLandscape()
  }, 100)
})

onUnmounted(() => {
  window.removeEventListener('resize', resizeCanvas)
  cancelAnimationFrame(animationFrameId)
})
</script>















<style scoped>
/* Container Match */
.game-container {
  display: flex;
  flex-direction: column;
  width: 100vw;
  height: 100vh;
  background-color: #0b0b0b; /* Pure, deep pitch black background */
  color: #ffffff;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  overflow: hidden;
  user-select: none;
  -webkit-user-select: none;
}

/* System Header Layout */
.game-header {
  flex-shrink: 0;
  padding: 40px 24px 16px 24px; /* Generous top padding mimicking native status bar spacing */
  text-align: center;
}

.game-header h1 {
  font-size: 1.75rem; /* Large, bold typography matching the 'About' header */
  font-weight: 700;
  letter-spacing: -0.02em;
  margin: 0 0 4px 0;
  color: #ffffff;
}

/* Sub-heading text styling for Version details if you choose to add them */
.game-header::after {
  display: block;
  font-size: 0.95rem;
  color: #666666; /* Muted gray text */
  margin-top: 4px;
  margin-bottom: 24px;
}

/* Mode Selector matching the floating UI panel block */
.mode-selector {
  display: flex;
  flex-direction: column; /* Stacks controls uniformly like list items */
  background: #121212; /* Distinct dark gray nested sheet background */
  border-radius: 16px; /* Smooth, rounded modern mobile corners */
  padding: 4px;
  margin: 0 auto;
  width: 90%;
  max-width: 400px;
}

.mode-selector button {
  background: transparent;
  border: none;
  color: #ffffff;
  padding: 18px 16px; /* Expanded touch targets matching list elements */
  font-size: 1rem;
  font-weight: 400;
  text-align: left; /* Clean left-aligned configuration */
  width: 100%;
  border-bottom: 1px solid #1c1c1c; /* Soft separation border rows */
  touch-action: manipulation;
  cursor: pointer;
  display: flex;
  justify-content: space-between;
}

/* Remove border line rule on the final item row inside the menu block */
.mode-selector button:last-child {
  border-bottom: none;
}

/* Highlighted state using clean contrast accents */
.mode-selector button.active {
  color: #4ade80; /* Subtle text accent or change background if preferred */
  font-weight: 600;
}

/* Clean Canvas Window Integration */
.viewport-wrapper {
  flex-grow: 1;
  flex-shrink: 1;
  min-height: 200px;
  position: relative;
  margin: 20px auto;
  width: 90%;
  max-width: 400px;
  background-color: #121212;
  border-radius: 16px; /* Encapsulates the canvas field inside an identical rounded card wrapper */
  overflow: hidden;
}

canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: block;
}

/* System Footer Layout matching the copyright imprint string position */
.game-footer {
  flex-shrink: 0;
  padding: 24px;
  text-align: center;
}

.game-footer p {
  font-size: 0.8rem;
  color: #444444; /* Low contrast deep muted gray text */
  margin: 0;
  letter-spacing: 0.01em;
}

</style>

