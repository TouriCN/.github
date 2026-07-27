# KeyView
<div id="keyview-root"></div>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, watch } from 'vue'

// 工具函数：读取CSS变量
const getCssVar = (name: string): string => {
  return getComputedStyle(document.documentElement).getPropertyValue(name).trim()
}

// 核心数据
const kps = ref(0)
const total = ref(0)
const showModal = ref(false)
const newKeyLabel = ref('')
const newKeyType = ref<'normal' | 'kps' | 'total'>('normal')
const nextKeyId = ref(7)

const rows = reactive([
  { id: 0, width: 52, color: '#ff3366' },
  { id: 1, width: 40, color: '#ff9f43' },
  { id: 2, width: 28, color: '#ffcc00' },
  { id: 3, width: 16, color: '#00ff88' }
])

const keys = reactive([
  { id: 1, label: 'F', code: 'KeyF', rowId: 0, cnt: 0, type: 'normal' as const, span: 1 },
  { id: 2, label: 'G', code: 'KeyG', rowId: 0, cnt: 0, type: 'normal' as const, span: 1 },
  { id: 3, label: 'H', code: 'KeyH', rowId: 0, cnt: 0, type: 'normal' as const, span: 1 },
  { id: 4, label: 'J', code: 'KeyJ', rowId: 0, cnt: 0, type: 'normal' as const, span: 1 },
  { id: 5, label: 'KPS', code: 'KPS_INFO', rowId: 1, cnt: 0, type: 'kps' as const, span: 2 },
  { id: 6, label: 'TOTAL', code: 'TOTAL_INFO', rowId: 1, cnt: 0, type: 'total' as const, span: 2 }
])

// DOM引用
const canvasRef = ref<HTMLCanvasElement>()
const kpsEl = ref<HTMLDivElement>()
const totalEl = ref<HTMLDivElement>()
const configKeysEl = ref<HTMLDivElement>()
const configRowsEl = ref<HTMLDivElement>()
const modalEl = ref<HTMLDivElement>()
const modalInputEl = ref<HTMLInputElement>()

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

// 更新主题色
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

// 计算按键位置
const updateKeyPositions = () => {
  if (!containerRect) return
  const baseWidth = 50
  const gap = 12
  const normalKeys = keys.filter(k => k.type === 'normal')
  const totalWidth = normalKeys.reduce((sum, k) => sum + baseWidth + (k.span - 1) * 56 + gap, 0) - gap
  let startX = (containerRect.width - totalWidth) / 2
  const keyY = 320 - 25 - 50
  normalKeys.forEach(key => {
    key.x = startX
    key.y = keyY
    key.w = baseWidth + (key.span - 1) * 56
    key.h = 50
    startX += key.w + gap
  })
}

// Canvas绘制
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
    ctx.fillRect(bar.x, 320 - bar.y - bar.height, bar.width, bar.height)
    if (bar.y < -bar.height) bars.splice(i, 1)
  }
}

// 按键逻辑
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

// 渲染配置区（纯DOM操作，不用Vue模板指令，避免解析错误）
const renderConfig = () => {
  if (!configKeysEl.value || !configRowsEl.value) return

  // 渲染按键配置
  configKeysEl.value.innerHTML = ''
  keys.forEach(key => {
    const div = document.createElement('div')
    div.className = 'kv-config-key'
    
    if (key.type === 'normal') {
      const delBtn = document.createElement('button')
      delBtn.className = 'kv-del-btn'
      delBtn.textContent = '×'
      delBtn.onclick = () => deleteKey(key.id)
      div.appendChild(delBtn)
    }

    const label = document.createElement('span')
    label.className = 'kv-key-label'
    label.textContent = key.label
    div.appendChild(label)

    const rowBtns = document.createElement('div')
    rowBtns.className = 'kv-row-btns'
    rows.forEach(row => {
      const btn = document.createElement('button')
      btn.className = `kv-row-btn ${key.rowId === row.id ? 'active' : ''}`
      btn.textContent = (row.id + 1).toString()
      btn.onclick = () => changeKeyRow(key.id, row.id)
      rowBtns.appendChild(btn)
    })
    div.appendChild(rowBtns)

    if (key.type === 'normal') {
      const spanBtns = document.createElement('div')
      spanBtns.className = 'kv-span-btns'
      ;[1,2,3,4].forEach(span => {
        const btn = document.createElement('button')
        btn.className = `kv-span-btn ${key.span === span ? 'active' : ''}`
        btn.textContent = span.toString()
        btn.onclick = () => changeKeySpan(key.id, span)
        spanBtns.appendChild(btn)
      })
      div.appendChild(spanBtns)
    }

    configKeysEl.value.appendChild(div)
  })

  // 添加按键按钮
  const addBtn = document.createElement('button')
  addBtn.className = 'kv-add-btn'
  addBtn.textContent = '+ 添加按键'
  addBtn.onclick = () => showModal.value = true
  configKeysEl.value.appendChild(addBtn)

  // 渲染行配置
  configRowsEl.value.innerHTML = ''
  rows.forEach(row => {
    const div = document.createElement('div')
    div.className = 'kv-config-row'

    const label = document.createElement('span')
    label.className = 'kv-row-label'
    label.style.color = row.color
    label.textContent = `第${row.id + 1}行`
    div.appendChild(label)

    const widthItem = document.createElement('div')
    widthItem.className = 'kv-row-config-item'
    widthItem.innerHTML = '<span>宽</span>'
    const widthInput = document.createElement('input')
    widthInput.type = 'range'
    widthInput.min = '8'
    widthInput.max = '56'
    widthInput.value = row.width.toString()
    widthInput.onchange = () => {
      row.width = parseInt(widthInput.value)
      saveConfig()
    }
    widthItem.appendChild(widthInput)
    div.appendChild(widthItem)

    const colorItem = document.createElement('div')
    colorItem.className = 'kv-row-config-item'
    colorItem.innerHTML = '<span>色</span>'
    const colorInput = document.createElement('input')
    colorInput.type = 'color'
    colorInput.value = row.color
    colorInput.onchange = () => {
      row.color = colorInput.value
      saveConfig()
    }
    colorItem.appendChild(colorInput)
    div.appendChild(colorItem)

    configRowsEl.value.appendChild(div)
  })
}

// 渲染模态框
const renderModal = () => {
  if (!modalEl.value) return
  
  if (showModal.value) {
    modalEl.value.innerHTML = `
      <div class="kv-modal-overlay" style="position:fixed;top:0;left:0;right:0;bottom:0;background:rgba(0,0,0,0.6);display:flex;align-items:center;justify-content:center;z-index:9999;" onclick="this.parentElement.style.display='none'">
        <div class="kv-modal" style="background:var(--vp-c-bg-elv);border:1px solid var(--vp-c-divider);border-radius:12px;padding:20px;width:300px;max-width:90vw;" onclick="event.stopPropagation()">
          <h3 style="margin:0 0 16px 0;color:var(--vp-c-text-1);font-size:16px;">添加按键</h3>
          <input type="text" placeholder="按键名（如 L）" style="width:100%;padding:8px 12px;border:1px solid var(--vp-c-divider);border-radius:6px;background:var(--vp-c-bg-soft);color:var(--vp-c-text-1);font-size:14px;margin-bottom:16px;" onkeydown="if(event.key==='Enter'){window.addKeyFromModal();}">
          <div style="display:flex;gap:8px;margin-bottom:20px;">
            <button class="modal-type-btn" data-type="normal" style="flex:1;padding:8px;border:1px solid var(--vp-c-divider);border-radius:6px;background:var(--vp-c-bg-soft);color:var(--vp-c-text-2);font-size:12px;cursor:pointer;">普通键</button>
            <button class="modal-type-btn" data-type="kps" style="flex:1;padding:8px;border:1px solid var(--vp-c-divider);border-radius:6px;background:var(--vp-c-bg-soft);color:var(--vp-c-text-2);font-size:12px;cursor:pointer;">KPS键</button>
            <button class="modal-type-btn" data-type="total" style="flex:1;padding:8px;border:1px solid var(--vp-c-divider);border-radius:6px;background:var(--vp-c-bg-soft);color:var(--vp-c-text-2);font-size:12px;cursor:pointer;">TOTAL键</button>
          </div>
          <div style="display:flex;gap:12px;">
            <button class="modal-cancel-btn" style="flex:1;padding:10px;border:none;border-radius:6px;background:var(--vp-c-bg-mute);color:var(--vp-c-text-1);cursor:pointer;font-weight:bold;">取消</button>
            <button class="modal-confirm-btn" style="flex:1;padding:10px;border:none;border-radius:6px;background:var(--vp-c-brand);color:white;cursor:pointer;font-weight:bold;">确定</button>
          </div>
        </div>
      </div>
    `
    // 绑定事件
    const input = modalEl.value.querySelector('input')
    if (input) {
      input.focus()
      modalInputEl.value = input
    }

    modalEl.value.querySelectorAll('.modal-type-btn').forEach(btn => {
      btn.addEventListener('click', (e) => {
        const target = e.target as HTMLElement
        const type = target.dataset.type as 'normal' | 'kps' | 'total'
        newKeyType.value = type
        modalEl.value!.querySelectorAll('.modal-type-btn').forEach(b => {
          (b as HTMLElement).style.borderColor = 'var(--vp-c-divider)'
          ;(b as HTMLElement).style.background = 'var(--vp-c-bg-soft)'
          ;(b as HTMLElement).style.color = 'var(--vp-c-text-2)'
        })
        target.style.borderColor = 'var(--vp-c-brand)'
        target.style.background = 'var(--vp-c-brand-dimm)'
        target.style.color = 'var(--vp-c-brand)'
      })
    })

    modalEl.value.querySelector('.modal-cancel-btn')?.addEventListener('click', () => {
      showModal.value = false
    })

    modalEl.value.querySelector('.modal-confirm-btn')?.addEventListener('click', () => {
      addNewKey()
    })

    // 默认选中普通键
    modalEl.value.querySelector('[data-type="normal"]')?.dispatchEvent(new Event('click'))
  } else {
    modalEl.value.innerHTML = ''
  }
}

// 暴露给全局的函数（供模态框内联调用）
;(window as any).addKeyFromModal = () => {
  const input = modalInputEl.value
  if (input) {
    newKeyLabel.value = input.value
    addNewKey()
  }
}

// 生命周期
onMounted(() => {
  const root = document.getElementById('keyview-root')
  if (!root) return

  // 创建DOM结构
  root.innerHTML = `
    <div class="kv-wrapper">
      <canvas id="kv-canvas" class="kv-canvas"></canvas>
      <div class="kv-info-bar">
        <div class="kv-kps" ref="kpsRef">KPS: 0</div>
        <div class="kv-total" ref="totalRef">TOTAL: 0</div>
        <button class="kv-reset">重置计数</button>
      </div>
      <div class="kv-config">
        <div class="kv-config-keys" ref="configKeysRef"></div>
        <div class="kv-config-rows" ref="configRowsRef"></div>
      </div>
      <div class="kv-modal-container" ref="modalRef"></div>
    </div>
  `

  // 获取DOM引用
  canvasRef.value = root.querySelector('#kv-canvas') as HTMLCanvasElement
  kpsEl.value = root.querySelector('.kv-kps') as HTMLDivElement
  totalEl.value = root.querySelector('.kv-total') as HTMLDivElement
  configKeysEl.value = root.querySelector('.kv-config-keys') as HTMLDivElement
  configRowsEl.value = root.querySelector('.kv-config-rows') as HTMLDivElement
  modalEl.value = root.querySelector('.kv-modal-container') as HTMLDivElement

  // 绑定重置按钮
  root.querySelector('.kv-reset')?.addEventListener('click', resetCounts)

  // 初始化Canvas
  ctx = canvasRef.value!.getContext('2d')
  containerRect = canvasRef.value!.parentElement!.getBoundingClientRect()
  dpr = window.devicePixelRatio || 1

  loadConfig()
  updateThemeColors()
  
  const initCanvas = () => {
    if (!canvasRef.value || !containerRect) return
    canvasRef.value.width = containerRect.width * dpr
    canvasRef.value.height = 320 * dpr
    canvasRef.value.style.width = `${containerRect.width}px`
    canvasRef.value.style.height = '320px'
    ctx?.scale(dpr, dpr)
    updateKeyPositions()
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
    // 更新KPS和TOTAL显示
    const now = Date.now()
    while (kpsRecords.length && kpsRecords[0] < now - 1000) kpsRecords.shift()
    kps.value = kpsRecords.length
    if (kpsEl.value) kpsEl.value.textContent = `KPS: ${kps.value}`
    if (totalEl.value) totalEl.value.textContent = `TOTAL: ${total.value}`
    requestAnimationFrame(animate)
  }
  animate()

  // 初始渲染配置
  renderConfig()

  // 监听数据变化，重新渲染配置
  watch([keys, rows], () => {
    renderConfig()
  }, { deep: true })

  watch(showModal, () => {
    renderModal()
  })

  // 清理
  onUnmounted(() => {
    observer.disconnect()
    document.removeEventListener('keydown', keydownHandler)
    document.removeEventListener('keyup', keyupHandler)
  })
})
</script>

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
  height: 320px;
  display: block;
}

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

.kv-modal-container {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 9999;
  display: none;
}

.kv-modal-container:has(.kv-modal-overlay) {
  display: block;
}
</style>

## 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。
