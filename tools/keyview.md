# KeyView

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, watch } from 'vue'

// 【工具函数：读取CSS变量】
const getCssVar = (name: string): string => {
  return getComputedStyle(document.documentElement).getPropertyValue(name).trim()
}

// 【状态层：核心数据】
const canvasRef = ref<HTMLCanvasElement>()
const kps = ref(0)
const total = ref(0)
const showModal = ref(false)
const newKeyLabel = ref('')
const newKeyType = ref<'normal' | 'kps' | 'total'>('normal')
const nextKeyId = ref(7)

// 行配置（对应上升条属性）
const rows = reactive([
  { id: 0, width: 52, color: '#ff3366' },
  { id: 1, width: 40, color: '#ff9f43' },
  { id: 2, width: 28, color: '#ffcc00' },
  { id: 3, width: 16, color: '#00ff88' }
])

// 按键配置
const keys = reactive([
  { id: 1, label: 'F', code: 'KeyF', rowId: 0, cnt: 0, type: 'normal' as const, span: 1 },
  { id: 2, label: 'G', code: 'KeyG', rowId: 0, cnt: 0, type: 'normal' as const, span: 1 },
  { id: 3, label: 'H', code: 'KeyH', rowId: 0, cnt: 0, type: 'normal' as const, span: 1 },
  { id: 4, label: 'J', code: 'KeyJ', rowId: 0, cnt: 0, type: 'normal' as const, span: 1 },
  { id: 5, label: 'KPS', code: 'KPS_INFO', rowId: 1, cnt: 0, type: 'kps' as const, span: 2 },
  { id: 6, label: 'TOTAL', code: 'TOTAL_INFO', rowId: 1, cnt: 0, type: 'total' as const, span: 2 }
])

// 运行时状态
const pressed = new Set<string>()
const bars: Array<{x:number,y:number,width:number,height:number,color:string}> = []
const kpsRecords: number[] = []
let dpr = 1
let ctx: CanvasRenderingContext2D | null = null
let containerRect: DOMRect | null = null
let themeColors = {
  bgSoft: '#f6f6f7',
  divider: '#e2e8f0',
  text1: '#1a202c',
  text3: '#64748b'
}
let isMounted = false // 标记DOM是否就绪

// 【工具方法层】
const updateThemeColors = () => {
  themeColors.bgSoft = getCssVar('--vp-c-bg-soft') || '#f6f6f7'
  themeColors.divider = getCssVar('--vp-c-divider') || '#e2e8f0'
  themeColors.text1 = getCssVar('--vp-c-text-1') || '#1a202c'
  themeColors.text3 = getCssVar('--vp-c-text-3') || '#64748b'
}

// 本地存储
const saveConfig = () => {
  try {
    localStorage.setItem('keyview_config', JSON.stringify({
      rows: rows.map(r => ({...r})),
      keys: keys.map(k => ({...k})),
      nextKeyId: nextKeyId.value
    }))
  } catch {}
}

const loadConfig = () => {
  try {
    const saved = localStorage.getItem('keyview_config')
    if (saved) {
      const data = JSON.parse(saved)
      rows.splice(0, rows.length, ...data.rows)
      keys.splice(0, keys.length, ...data.keys)
      nextKeyId.value = data.nextKeyId || 7
      total.value = keys.filter(k => k.type === 'normal').reduce((sum, k) => sum + k.cnt, 0)
    }
  } catch {}
}

// 计算按键位置（严格判空，避免时序报错）
const updateKeyPositions = () => {
  if (!isMounted || !containerRect) return // 只有DOM就绪后才执行
  const baseWidth = 50
  const gap = 12
  const normalKeys = keys.filter(k => k.type === 'normal')
  const totalWidth = normalKeys.reduce((sum, k) => sum + baseWidth + (k.span - 1) * 56 + gap, 0) - gap
  let startX = (containerRect.width - totalWidth) / 2
  const keyY = 520 - 200 - 25 - 50
  normalKeys.forEach(key => {
    key.x = startX
    key.y = keyY
    key.w = baseWidth + (key.span - 1) * 56
    key.h = 50
    startX += key.w + gap
  })
}

// Canvas绘制逻辑
const drawKeys = () => {
  if (!ctx || !containerRect) return
  const normalKeys = keys.filter(k => k.type === 'normal')
  normalKeys.forEach(key => {
    ctx.fillStyle = themeColors.bgSoft
    ctx.fillRect(key.x, key.y, key.w, key.h)
    ctx.lineWidth = 2
    const row = rows.find(r => r.id === key.rowId)!
    ctx.strokeStyle = pressed.has(key.code) ? row.color : themeColors.divider
    ctx.strokeRect(key.x, key.y, key.w, key.h)
    if (pressed.has(key.code)) {
      ctx.save()
      ctx.translate(key.x + key.w/2, key.y + key.h/2)
      ctx.scale(0.96, 0.96)
      ctx.translate(-(key.x + key.w/2), -(key.y + key.h/2))
    }
    ctx.fillStyle = themeColors.text1
    ctx.font = 'bold 15px -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(key.label, key.x + key.w/2, key.y + key.h/2 - 6)
    ctx.font = '10px -apple-system,BlinkMacSystemFont,"Segoe UI",Roboto'
    ctx.fillStyle = themeColors.text3
    ctx.fillText(key.cnt.toString(), key.x + key.w/2, key.y + key.h/2 + 10)
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
    if (key.type === 'normal') {
      key.cnt++
      total.value++
      const row = rows.find(r => r.id === key.rowId)!
      const keyEl = keys.find(k => k.code === code)!
      bars.push({
        x: keyEl.x + (keyEl.w - row.width) / 2,
        y: 0,
        width: row.width,
        height: 2,
        color: row.color
      })
      saveConfig()
    }
  }
}

const handleUp = (code: string) => pressed.delete(code)

const resetCounts = () => {
  keys.forEach(key => {
    if (key.type === 'normal') key.cnt = 0
  })
  kpsRecords.length = 0
  kps.value = 0
  total.value = 0
  saveConfig()
}

// 配置操作
const deleteKey = (id: number) => {
  const index = keys.findIndex(k => k.id === id)
  if (index > -1) {
    keys.splice(index, 1)
    saveConfig()
  }
}

const changeKeyRow = (id: number, rowId: number) => {
  const key = keys.find(k => k.id === id)
  if (key) {
    key.rowId = rowId
    saveConfig()
  }
}

const changeKeySpan = (id: number, span: number) => {
  const key = keys.find(k => k.id === id)
  if (key && key.type === 'normal') {
    key.span = span
    saveConfig()
  }
}

const addNewKey = () => {
  const label = newKeyLabel.value.trim() || 'N'
  const code = newKeyType.value === 'normal' ? `Key${label.toUpperCase()}` : `INFO_${nextKeyId.value}`
  keys.push({
    id: nextKeyId.value++,
    label,
    code,
    rowId: 0,
    cnt: 0,
    type: newKeyType.value,
    span: 1
  })
  newKeyLabel.value = ''
  newKeyType.value = 'normal'
  showModal.value = false
  saveConfig()
}

// 【生命周期层】
onMounted(() => {
  isMounted = true // DOM就绪标记
  if (!canvasRef.value) return
  ctx = canvasRef.value.getContext('2d')
  containerRect = canvasRef.value.parentElement!.getBoundingClientRect()
  dpr = window.devicePixelRatio || 1

  // 初始化
  loadConfig()
  updateThemeColors()
  const initCanvas = () => {
    if (!canvasRef.value || !containerRect) return
    canvasRef.value.width = containerRect.width * dpr
    canvasRef.value.height = 520 * dpr
    canvasRef.value.style.width = `${containerRect.width}px`
    canvasRef.value.style.height = '320px' // 显式设置高度，避免calc计算问题
    ctx?.scale(dpr, dpr)
    updateKeyPositions() // 初始化后计算位置
  }
  initCanvas()

  // 监听主题变化
  const observer = new MutationObserver(updateThemeColors)
  observer.observe(document.documentElement, { attributes: true, attributeFilter: ['class'] })

  // 窗口resize
  window.addEventListener('resize', () => {
    if (!canvasRef.value) return
    containerRect = canvasRef.value.parentElement!.getBoundingClientRect()
    initCanvas()
  })

  // 键盘事件
  const keydownHandler = (e: KeyboardEvent) => {
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName || '')) return
    const key = keys.find(k => k.code === e.code && k.type === 'normal')
    if (key) handleDown(e.code)
  }
  const keyupHandler = (e: KeyboardEvent) => {
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName || '')) return
    const key = keys.find(k => k.code === e.code && k.type === 'normal')
    if (key) handleUp(e.code)
  }
  document.addEventListener('keydown', keydownHandler)
  document.addEventListener('keyup', keyupHandler)
  window.addEventListener('blur', () => pressed.clear())

  // 动画循环
  const animate = () => {
    if (!ctx || !containerRect) return
    ctx.clearRect(0, 0, containerRect.width, 320)
    drawBars()
    drawKeys()
    const now = Date.now()
    while (kpsRecords.length && kpsRecords[0] < now - 1000) kpsRecords.shift()
    kps.value = kpsRecords.length
    requestAnimationFrame(animate)
  }
  animate()

  // 清理
  onUnmounted(() => {
    isMounted = false
    observer.disconnect()
    document.removeEventListener('keydown', keydownHandler)
    document.removeEventListener('keyup', keyupHandler)
  })
})

// 【关键修复】watch延迟到DOM更新后执行，避免时序报错
watch(keys, updateKeyPositions, { deep: true, flush: 'post' })
</script>

<template>
  <div class="kv-wrapper">
    <!-- Canvas绘制区域 -->
    <canvas ref="canvasRef" class="kv-canvas"></canvas>
    
    <!-- 顶部信息栏 -->
    <div class="kv-info-bar">
      <div class="kv-kps">KPS: {{ kps }}</div>
      <div class="kv-total">TOTAL: {{ total }}</div>
      <button class="kv-reset" @click="resetCounts">重置计数</button>
    </div>

    <!-- 配置区域 -->
    <div class="kv-config">
      <!-- 按键配置 -->
      <div class="kv-config-keys">
        <div v-for="key in keys" :key="key.id" class="kv-config-key">
          <button class="kv-del-btn" @click="deleteKey(key.id)" v-if="key.type === 'normal'">×</button>
          <span class="kv-key-label">{{ key.label }}</span>
          <div class="kv-row-btns">
            <button 
              v-for="row in rows" 
              :key="row.id"
              class="kv-row-btn"
              :class="{ active: key.rowId === row.id }"
              @click="changeKeyRow(key.id, row.id)"
            >{{ row.id + 1 }}</button>
          </div>
          <div class="kv-span-btns" v-if="key.type === 'normal'">
            <button 
              v-for="span in [1,2,3,4]" 
              :key="span"
              class="kv-span-btn"
              :class="{ active: key.span === span }"
              @click="changeKeySpan(key.id, span)"
            >{{ span }}</button>
          </div>
        </div>
        <button class="kv-add-btn" @click="showModal = true">+ 添加按键</button>
      </div>

      <!-- 行配置 -->
      <div class="kv-config-rows">
        <div v-for="row in rows" :key="row.id" class="kv-config-row">
          <span class="kv-row-label" :style="{ color: row.color }">第{{ row.id + 1 }}行</span>
          <div class="kv-row-config-item">
            <span>宽</span>
            <input 
              type="range" 
              min="8" 
              max="56" 
              v-model.number="row.width"
              @change="saveConfig"
            >
          </div>
          <div class="kv-row-config-item">
            <span>色</span>
            <input 
              type="color" 
              v-model="row.color"
              @change="saveConfig"
            >
          </div>
        </div>
      </div>
    </div>

    <!-- 添加按键弹窗 -->
    <div class="kv-modal-overlay" v-if="showModal" @click.self="showModal = false">
      <div class="kv-modal">
        <h3>添加按键</h3>
        <input 
          type="text" 
          v-model="newKeyLabel"
          placeholder="按键名（如 L）"
          @keyup.enter="addNewKey"
        >
        <div class="kv-modal-types">
          <button 
            class="kv-type-btn"
            :class="{ active: newKeyType === 'normal' }"
            @click="newKeyType = 'normal'"
          >普通键</button>
          <button 
            class="kv-type-btn"
            :class="{ active: newKeyType === 'kps' }"
            @click="newKeyType = 'kps'"
          >KPS键</button>
          <button 
            class="kv-type-btn"
            :class="{ active: newKeyType === 'total' }"
            @click="newKeyType = 'total'"
          >TOTAL键</button>
        </div>
        <div class="kv-modal-actions">
          <button class="kv-cancel-btn" @click="showModal = false">取消</button>
          <button class="kv-confirm-btn" @click="addNewKey">确定</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
.kv-wrapper {
  position: relative;
  width: 100%;
  margin: 24px 0;
  height: 520px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 8px;
  overflow: hidden;
  background: var(--vp-c-bg);
}

.kv-canvas {
  width: 100%;
  height: 320px; /* 显式高度，避免calc问题 */
  display: block;
}

/* 顶部信息栏 */
.kv-info-bar {
  position: absolute;
  top: 10px;
  left: 10px;
  right: 10px;
  display: flex;
  gap: 12px;
  align-items: center;
}

.kv-kps, .kv-total {
  padding: 4px 8px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 4px;
  background: var(--vp-c-bg-soft);
  color: var(--vp-c-text-1);
  font-size: 12px;
  font-weight: bold;
}

.kv-reset {
  margin-left: auto;
  padding: 4px 8px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 4px;
  background: var(--vp-c-bg-soft);
  color: var(--vp-c-text-2);
  font-size: 12px;
  cursor: pointer;
}

/* 配置区域 */
.kv-config {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 200px;
  background: var(--vp-c-bg-soft);
  border-top: 1px solid var(--vp-c-divider);
  padding: 12px;
  display: flex;
  gap: 16px;
  overflow: auto;
}

.kv-config-keys {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  align-content: flex-start;
}

.kv-config-key {
  position: relative;
  padding: 8px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 6px;
  background: var(--vp-c-bg-mute);
  display: flex;
  flex-direction: column;
  gap: 6px;
  min-width: 80px;
}

.kv-del-btn {
  position: absolute;
  top: -6px;
  right: -6px;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: var(--vp-c-danger);
  color: white;
  border: none;
  cursor: pointer;
  font-size: 10px;
  line-height: 16px;
}

.kv-key-label {
  font-size: 13px;
  color: var(--vp-c-text-1);
  text-align: center;
}

.kv-row-btns, .kv-span-btns {
  display: flex;
  gap: 2px;
}

.kv-row-btn, .kv-span-btn {
  width: 18px;
  height: 18px;
  border-radius: 4px;
  border: 1px solid var(--vp-c-divider);
  background: var(--vp-c-bg-soft);
  color: var(--vp-c-text-2);
  font-size: 10px;
  cursor: pointer;
}

.kv-row-btn.active, .kv-span-btn.active {
  border-color: var(--vp-c-brand);
  background: var(--vp-c-brand-dimm);
  color: var(--vp-c-brand);
}

.kv-add-btn {
  padding: 8px 12px;
  border: 2px dashed var(--vp-c-divider);
  border-radius: 6px;
  background: transparent;
  color: var(--vp-c-text-2);
  cursor: pointer;
  align-self: flex-start;
}

/* 行配置 */
.kv-config-rows {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  align-content: flex-start;
}

.kv-config-row {
  padding: 8px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 6px;
  background: var(--vp-c-bg-mute);
  display: flex;
  flex-direction: column;
  gap: 6px;
  min-width: 100px;
}

.kv-row-label {
  font-size: 12px;
  font-weight: bold;
}

.kv-row-config-item {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 11px;
  color: var(--vp-c-text-3);
}

.kv-row-config-item input[type="range"] {
  width: 60px;
}

.kv-row-config-item input[type="color"] {
  width: 24px;
  height: 24px;
  border: none;
  background: none;
}

/* 弹窗 */
.kv-modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.6);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
}

.kv-modal {
  background: var(--vp-c-bg-elv);
  border: 1px solid var(--vp-c-divider);
  border-radius: 12px;
  padding: 20px;
  width: 300px;
  max-width: 90vw;
}

.kv-modal h3 {
  margin: 0 0 16px 0;
  color: var(--vp-c-text-1);
  font-size: 16px;
}

.kv-modal input[type="text"] {
  width: 100%;
  padding: 8px 12px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 6px;
  background: var(--vp-c-bg-soft);
  color: var(--vp-c-text-1);
  font-size: 14px;
  margin-bottom: 16px;
}

.kv-modal-types {
  display: flex;
  gap: 8px;
  margin-bottom: 20px;
}

.kv-type-btn {
  flex: 1;
  padding: 8px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 6px;
  background: var(--vp-c-bg-soft);
  color: var(--vp-c-text-2);
  font-size: 12px;
  cursor: pointer;
}

.kv-type-btn.active {
  border-color: var(--vp-c-brand);
  background: var(--vp-c-brand-dimm);
  color: var(--vp-c-brand);
}

.kv-modal-actions {
  display: flex;
  gap: 12px;
}

.kv-cancel-btn, .kv-confirm-btn {
  flex: 1;
  padding: 10px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
}

.kv-cancel-btn {
  background: var(--vp-c-bg-mute);
  color: var(--vp-c-text-1);
}

.kv-confirm-btn {
  background: var(--vp-c-brand);
  color: white;
}
</style>

## 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。
