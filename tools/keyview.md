# KeyView

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted } from 'vue'

// 【工具函数：读取CSS变量，解决canvas不兼容CSS变量的问题】
const getCssVar = (name: string): string => {
  return getComputedStyle(document.documentElement).getPropertyValue(name).trim()
}

// 【状态层】
const canvasRef = ref<HTMLCanvasElement>()
const kps = ref(0)
const keys = reactive([
  { code: 'KeyF', label: 'F', cnt: 0, color: '#ff3366', x: 0, y: 0, w: 50, h: 50 },
  { code: 'KeyG', label: 'G', cnt: 0, color: '#ff9f43', x: 0, y: 0, w: 50, h: 50 },
  { code: 'KeyH', label: 'H', cnt: 0, color: '#ffcc00', x: 0, y: 0, w: 50, h: 50 },
  { code: 'KeyJ', label: 'J', cnt: 0, color: '#00ff88', x: 0, y: 0, w: 50, h: 50 }
])
const pressed = new Set<string>()
const bars: Array<{x:number,y:number,width:number,height:number,color:string}> = []
const kpsRecords: number[] = []
let dpr = 1
let ctx: CanvasRenderingContext2D | null = null
let containerRect: DOMRect | null = null
// 缓存主题相关颜色，避免每次绘制都读DOM
let themeColors = {
  bgSoft: '',
  divider: '',
  text1: '',
  text3: ''
}

// 【工具方法层】
const updateThemeColors = () => {
  themeColors.bgSoft = getCssVar('--vp-c-bg-soft') || '#f6f6f7'
  themeColors.divider = getCssVar('--vp-c-divider') || '#e2e8f0'
  themeColors.text1 = getCssVar('--vp-c-text-1') || '#1a202c'
  themeColors.text3 = getCssVar('--vp-c-text-3') || '#64748b'
}

const updateKeyPositions = () => {
  if (!containerRect) return
  const keyW = 50
  const gap = 12
  const totalW = keys.length * keyW + (keys.length - 1) * gap
  let startX = (containerRect.width - totalW) / 2
  const keyY = 520 - 200 - 25 - keyW
  keys.forEach(key => {
    key.x = startX
    key.y = keyY
    startX += keyW + gap
  })
}

const drawKeys = () => {
  if (!ctx || !containerRect) return
  keys.forEach(key => {
    // 按键底色
    ctx.fillStyle = themeColors.bgSoft
    ctx.fillRect(key.x, key.y, key.w, key.h)
    // 边框
    ctx.lineWidth = 2
    ctx.strokeStyle = pressed.has(key.code) ? key.color : themeColors.divider
    ctx.strokeRect(key.x, key.y, key.w, key.h)
    // 按下缩放反馈
    if (pressed.has(key.code)) {
      ctx.save()
      ctx.translate(key.x + key.w/2, key.y + key.h/2)
      ctx.scale(0.96, 0.96)
      ctx.translate(-(key.x + key.w/2), -(key.y + key.h/2))
    }
    // 按键文字
    ctx.fillStyle = themeColors.text1
    ctx.font = 'bold 15px -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(key.label, key.x + key.w/2, key.y + key.h/2 - 6)
    // 计数文字
    ctx.font = '10px -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto'
    ctx.fillStyle = themeColors.text3
    ctx.fillText(key.cnt, key.x + key.w/2, key.y + key.h/2 + 10)
    if (pressed.has(key.code)) ctx.restore()
  })
}

const drawBars = () => {
  if (!ctx) return
  for (let i = bars.length - 1; i >= 0; i--) {
    const bar = bars[i]
    bar.y -= 2.5
    ctx.fillStyle = bar.color
    ctx.fillRect(bar.x, 520 - 200 - bar.y - bar.height, bar.width, bar.height)
    if (bar.y < -bar.height) bars.splice(i, 1)
  }
}

// 【业务逻辑层】
const handleDown = (code: string) => {
  if (pressed.has(code)) return
  pressed.add(code)
  kpsRecords.push(Date.now())
  const key = keys.find(k => k.code === code)
  if (key) {
    key.cnt++
    bars.push({ x: key.x, y: 0, width: key.w, height: 2, color: key.color })
  }
}

const handleUp = (code: string) => pressed.delete(code)

const resetCounts = () => {
  keys.forEach(key => key.cnt = 0)
  kpsRecords.length = 0
  kps.value = 0
}

// 【生命周期层】
onMounted(() => {
  if (!canvasRef.value) return
  ctx = canvasRef.value.getContext('2d')
  containerRect = canvasRef.value.parentElement!.getBoundingClientRect()
  dpr = window.devicePixelRatio || 1

  // 初始化主题颜色和画布
  updateThemeColors()
  const initCanvas = () => {
    if (!canvasRef.value || !containerRect) return
    canvasRef.value.width = containerRect.width * dpr
    canvasRef.value.height = 520 * dpr
    canvasRef.value.style.width = `${containerRect.width}px`
    canvasRef.value.style.height = '520px'
    ctx?.scale(dpr, dpr)
    updateKeyPositions()
  }
  initCanvas()

  // 监听主题变化，更新颜色缓存
  const observer = new MutationObserver(() => updateThemeColors())
  observer.observe(document.documentElement, { attributes: true, attributeFilter: ['class'] })

  // 窗口 resize 适配
  window.addEventListener('resize', () => {
    if (!canvasRef.value) return
    containerRect = canvasRef.value.parentElement!.getBoundingClientRect()
    initCanvas()
  })

  // 键盘事件
  const keydownHandler = (e: KeyboardEvent) => {
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName || '')) return
    if (keys.some(k => k.code === e.code)) handleDown(e.code)
  }
  const keyupHandler = (e: KeyboardEvent) => {
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName || '')) return
    if (keys.some(k => k.code === e.code)) handleUp(e.code)
  }
  document.addEventListener('keydown', keydownHandler)
  document.addEventListener('keyup', keyupHandler)
  window.addEventListener('blur', () => pressed.clear())

  // 动画循环
  const animate = () => {
    if (!ctx || !containerRect) return
    ctx.clearRect(0, 0, containerRect.width, 520)
    drawBars()
    drawKeys()
    // 更新KPS
    const now = Date.now()
    while (kpsRecords.length && kpsRecords[0] < now - 1000) kpsRecords.shift()
    kps.value = kpsRecords.length
    requestAnimationFrame(animate)
  }
  animate()

  // 清理
  onUnmounted(() => {
    observer.disconnect()
    document.removeEventListener('keydown', keydownHandler)
    document.removeEventListener('keyup', keyupHandler)
  })
})
</script>

<template>
  <div class="kv-wrapper">
    <canvas ref="canvasRef" class="kv-canvas"></canvas>
    <button class="kv-reset" @click="resetCounts">重置计数</button>
    <div class="kv-kps">KPS: {{ kps }}</div>
  </div>
</template>

<style>
.kv-wrapper {
  position: relative;
  width: 100%;
  margin: 24px 0;
}
.kv-canvas {
  width: 100%;
  height: 520px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 8px;
  display: block;
  /* 显式设置canvas默认背景，避免透明导致的显示异常 */
  background: var(--vp-c-bg);
}
.kv-reset {
  position: absolute;
  top: 32px;
  right: 10px;
  padding: 4px 8px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 4px;
  background: var(--vp-c-bg-soft);
  color: var(--vp-c-text-2);
  font-size: 12px;
  cursor: pointer;
}
.kv-kps {
  position: absolute;
  top: 32px;
  left: 10px;
  padding: 4px 8px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 4px;
  background: var(--vp-c-bg-soft);
  color: var(--vp-c-text-1);
  font-size: 12px;
  font-weight: bold;
}
</style>

## 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。
