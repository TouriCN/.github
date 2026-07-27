# KeyView

<!-- 内联Vue组件：VitePress原生支持的架构，不用拆文件，逻辑自洽 -->
<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted } from 'vue'

// 【架构分层：状态层】所有响应式数据集中管理，不用全局变量
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

// 【架构分层：工具方法层】纯函数，无副作用，方便维护
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
  const isDark = document.documentElement.classList.contains('dark')
  keys.forEach(key => {
    ctx.fillStyle = isDark ? 'var(--vp-c-bg-soft)' : '#f6f6f7'
    ctx.fillRect(key.x, key.y, key.w, key.h)
    ctx.lineWidth = 2
    ctx.strokeStyle = pressed.has(key.code) ? key.color : 'var(--vp-c-divider)'
    ctx.strokeRect(key.x, key.y, key.w, key.h)
    if (pressed.has(key.code)) {
      ctx.save()
      ctx.translate(key.x + key.w/2, key.y + key.h/2)
      ctx.scale(0.96, 0.96)
      ctx.translate(-(key.x + key.w/2), -(key.y + key.h/2))
    }
    ctx.fillStyle = 'var(--vp-c-text-1)'
    ctx.font = 'bold 15px -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(key.label, key.x + key.w/2, key.y + key.h/2 - 6)
    ctx.font = '10px -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto'
    ctx.fillStyle = 'var(--vp-c-text-3)'
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

// 【架构分层：业务逻辑层】事件处理、KPS计算等核心逻辑
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

// 【架构分层：生命周期层】Vue原生钩子，天然只在客户端执行，不用手动写SSR判断
onMounted(() => {
  if (!canvasRef.value) return
  ctx = canvasRef.value.getContext('2d')
  containerRect = canvasRef.value.parentElement!.getBoundingClientRect()
  dpr = window.devicePixelRatio || 1

  // 初始化画布
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
  window.addEventListener('resize', initCanvas)

  // 键盘事件绑定
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

  // 清理事件
  onUnmounted(() => {
    document.removeEventListener('keydown', keydownHandler)
    document.removeEventListener('keyup', keyupHandler)
  })
})
</script>

<!-- 【架构分层：视图层】模板和逻辑完全解耦，没有内联JS -->
<template>
  <div class="kv-wrapper">
    <canvas ref="canvasRef" class="kv-canvas"></canvas>
    <button class="kv-reset" @click="resetCounts">重置计数</button>
    <div class="kv-kps">KPS: {{ kps }}</div>
  </div>
</template>

<!-- 【架构分层：样式层】样式和逻辑解耦，用VitePress主题变量 -->
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
