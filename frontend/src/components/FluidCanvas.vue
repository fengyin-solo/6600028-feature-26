<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch, nextTick } from 'vue'
import { useFluidStore } from '../store/fluid'

const store = useFluidStore()
const canvas = ref<HTMLCanvasElement | null>(null)
const container = ref<HTMLDivElement | null>(null)

const canvasWidth = ref(800)
const canvasHeight = ref(500)
const dpr = ref(window.devicePixelRatio || 1)

let resizeObserver: ResizeObserver | null = null
let resizeTimeout: number | null = null

function velocityToColor(speed: number): string {
  const maxSpeed = 200
  const t = Math.min(speed / maxSpeed, 1)
  const hue = (1 - t) * 240
  const sat = 80
  const light = 40 + t * 20
  return `hsl(${hue}, ${sat}%, ${light}%)`
}

function draw() {
  const ctx = canvas.value?.getContext('2d')
  if (!ctx) return

  const W = canvasWidth.value
  const H = canvasHeight.value

  ctx.fillStyle = '#0c1222'
  ctx.fillRect(0, 0, W, H)

  ctx.strokeStyle = '#475569'
  ctx.lineWidth = 3
  ctx.strokeRect(2, 2, W - 4, H - 4)

  ctx.strokeStyle = '#1e293b'
  ctx.lineWidth = 0.3
  for (let x = 0; x < W; x += 50) {
    ctx.beginPath()
    ctx.moveTo(x, 0)
    ctx.lineTo(x, H)
    ctx.stroke()
  }
  for (let y = 0; y < H; y += 50) {
    ctx.beginPath()
    ctx.moveTo(0, y)
    ctx.lineTo(W, y)
    ctx.stroke()
  }

  if (!store.engine) return

  const gridSize = 20
  const gw = Math.ceil(W / gridSize)
  const gh = Math.ceil(H / gridSize)
  const densityGrid = new Float32Array(gw * gh)
  for (const p of store.engine.particles) {
    const gx = Math.floor(p.x / gridSize)
    const gy = Math.floor(p.y / gridSize)
    if (gx >= 0 && gx < gw && gy >= 0 && gy < gh) {
      densityGrid[gy * gw + gx] += p.density
    }
  }
  const maxDens = Math.max(...densityGrid, 1)
  for (let gy = 0; gy < gh; gy++) {
    for (let gx = 0; gx < gw; gx++) {
      const d = densityGrid[gy * gw + gx]
      if (d > 0) {
        const alpha = Math.min(d / maxDens * 0.15, 0.15)
        ctx.fillStyle = `rgba(59, 130, 246, ${alpha})`
        ctx.fillRect(gx * gridSize, gy * gridSize, gridSize, gridSize)
      }
    }
  }

  const particles = store.engine.particles
  for (const p of particles) {
    const speed = Math.sqrt(p.vx * p.vx + p.vy * p.vy)
    const color = velocityToColor(speed)
    const radius = 4

    ctx.beginPath()
    ctx.arc(p.x, p.y, radius, 0, Math.PI * 2)
    ctx.fillStyle = color
    ctx.fill()
  }

  ctx.fillStyle = 'rgba(0, 0, 0, 0.6)'
  ctx.fillRect(W - 80, 5, 75, 22)
  ctx.fillStyle = '#22c55e'
  ctx.font = 'bold 12px monospace'
  ctx.fillText(`FPS: ${store.fps}`, W - 74, 20)

  ctx.fillStyle = 'rgba(0, 0, 0, 0.6)'
  ctx.fillRect(W - 120, 30, 115, 22)
  ctx.fillStyle = '#94a3b8'
  ctx.font = '11px monospace'
  ctx.fillText(`Frame: ${store.frameCount}`, W - 114, 44)
}

let raf: number | null = null
function animate() {
  draw()
  raf = requestAnimationFrame(animate)
}

function onClick(e: MouseEvent) {
  if (!store.engine || !canvas.value) return
  const rect = canvas.value.getBoundingClientRect()
  const scaleX = canvasWidth.value / rect.width
  const scaleY = canvasHeight.value / rect.height
  const x = (e.clientX - rect.left) * scaleX
  const y = (e.clientY - rect.top) * scaleY
  store.engine.applyImpulse(x, y, 300)
}

function handleResize() {
  if (!container.value) return
  const rect = container.value.getBoundingClientRect()
  const newWidth = Math.floor(rect.width)
  const newHeight = Math.floor(rect.height)

  if (newWidth <= 0 || newHeight <= 0) return
  if (newWidth === canvasWidth.value && newHeight === canvasHeight.value) return

  canvasWidth.value = newWidth
  canvasHeight.value = newHeight
  dpr.value = window.devicePixelRatio || 1

  if (canvas.value) {
    canvas.value.width = newWidth * dpr.value
    canvas.value.height = newHeight * dpr.value
    const ctx = canvas.value.getContext('2d')
    if (ctx) {
      ctx.setTransform(dpr.value, 0, 0, dpr.value, 0, 0)
    }
  }

  store.resize(newWidth, newHeight)
}

function onContainerResize() {
  if (resizeTimeout !== null) {
    clearTimeout(resizeTimeout)
  }
  resizeTimeout = window.setTimeout(() => {
    handleResize()
  }, 50)
}

onMounted(() => {
  if (container.value) {
    resizeObserver = new ResizeObserver(onContainerResize)
    resizeObserver.observe(container.value)
  }
  nextTick(() => {
    handleResize()
    animate()
  })
})

onUnmounted(() => {
  if (raf) cancelAnimationFrame(raf)
  if (resizeObserver) resizeObserver.disconnect()
  if (resizeTimeout !== null) clearTimeout(resizeTimeout)
})
</script>

<template>
  <div ref="container" class="w-full flex-1 min-h-0">
    <canvas
      ref="canvas"
      class="rounded-lg border border-gray-700 cursor-crosshair w-full h-full block"
      @click="onClick"
    />
  </div>
</template>
