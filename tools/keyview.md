# KeyView
<div class="kv-wrapper">
  <!-- 静态Canvas，无动态指令 -->
  <canvas id="kv-canvas" class="kv-canvas"></canvas>
  
  <!-- 静态信息栏，数值用JS更新 -->
  <div class="kv-info-bar">
    <div class="kv-kps">KPS: <span id="kps-num">0</span></div>
    <div class="kv-total">TOTAL: <span id="total-num">0</span></div>
    <button class="kv-reset" id="reset-btn">重置计数</button>
  </div>

  <!-- 静态容器，按键和配置都用JS动态插 -->
  <div class="kv-config">
    <div class="kv-config-keys" id="keys-container"></div>
    <div class="kv-config-rows" id="rows-container"></div>
  </div>

  <!-- 模态框容器，JS控制显示 -->
  <div id="modal-root"></div>
</div>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, watch } from 'vue'

// 工具函数：读CSS变量
const getCssVar = (name: string): string => {
  return getComputedStyle(document.documentElement).getPropertyValue(name).trim()
}

// 核心数据（和之前完全一致）
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
  { id: 1, label: 'F', code: 'KeyF', rowId: 0, cnt: 0, type: 'normal' as const, span: 1, x: 0, y: 0, w: 0, h: 0 },
  { id: 2, label: 'G', code: 'KeyG', rowId: 0, cnt: 0, type: 'normal' as const, span: 1, x: 0, y: 0, w: 0, h: 0 },
  { id: 3, label: 'H', code: 'KeyH', rowId: 0, cnt: 0, type: 'normal' as const, span: 1, x: 0, y: 0, w: 0, h: 0 },
  { id: 4, label: 'J', code: 'KeyJ', rowId: 0, cnt: 0, type: 'normal' as const, span: 1, x: 0, y: 0, w: 0, h: 0 },
  { id: 5, label: 'KPS', code: 'KPS_INFO', rowId: 1, cnt: 0, type: 'kps' as const, span: 2 },
  { id: 6, label: 'TOTAL', code: 'TOTAL_INFO', rowId: 1, cnt: 0, type: 'total' as const, span: 2 }
])

// Canvas相关
let canvas: HTMLCanvasElement | null = null
let ctx: CanvasRenderingContext2D | null = null
let containerRect: DOMRect | null = null
let dpr = 1
const pressed = new Set<string>()
const bars: Array<{x:number,y:number,width:number,height:number,color:string}> = []
const kpsRecords: number[] = []
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
      keys: keys.map(k => ({...k, x: 0, y: 0, w: 0, h: 0})), // 去掉临时坐标再存
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
      keys.splice(0, keys.length, ...data.keys.map((k: any) => ({...k, x: 0, y: 0, w: 0, h: 0})))
      nextKeyId.value = data.nextKeyId || 7
      total.value = keys.filter(k => k.type === 'normal').reduce((sum, k) => sum + k.cnt, 0)
    }
  } catch {}
}

// 计算按键位置（纯计算，不碰DOM）
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
  // 更新显示
  document.getElementById('kps-num')!.textContent = '0'
  document.getElementById('total-num')!.textContent = '0'
  renderKeys() // 重绘按键上的计数
}

// ====== 核心：用JS动态生成DOM，不用Vue指令，避免解析错误 ======
// 渲染按键配置区
const renderKeys = () => {
  const container = document.getElementById('keys-container')
  if (!container) return
  container.innerHTML = ''

  keys.forEach(key => {
    const keyDiv = document.createElement('div')
    keyDiv.className = 'kv-config-key'

    // 删除按钮（仅普通键）
    if (key.type === 'normal') {
      const delBtn = document.createElement('button')
      delBtn.className = 'kv-del-btn'
      delBtn.textContent = '×'
      delBtn.onclick = () => {
        const idx = keys.findIndex(k => k.id === key.id)
        if (idx > -1) {
          keys.splice(idx, 1)
          saveConfig()
          renderKeys()
        }
      }
      keyDiv.appendChild(delBtn)
    }

    // 按键标签
    const label = document.createElement('span')
    label.className = 'kv-key-label'
    label.textContent = key.label
    keyDiv.appendChild(label)

    // 行选择按钮
    const rowBtns = document.createElement('div')
    rowBtns.className = 'kv-row-btns'
    rows.forEach(row => {
      const btn = document.createElement('button')
      btn.className = `kv-row-btn ${key.rowId === row.id ? 'active' : ''}`
      btn.textContent = (row.id + 1).toString()
      btn.onclick = () => {
        key.rowId = row.id
        saveConfig()
        renderKeys()
      }
      rowBtns.appendChild(btn)
    })
    keyDiv.appendChild(rowBtns)

    // 跨度选择按钮（仅普通键）
    if (key.type === 'normal') {
      const spanBtns = document.createElement('div')
      spanBtns.className = 'kv-span-btns'
      ;[1,2,3,4].forEach(span => {
        const btn = document.createElement('button')
        btn.className = `kv-span-btn ${key.span === span ? 'active' : ''}`
        btn.textContent = span.toString()
        btn.onclick = () => {
          key.span = span
          saveConfig()
          updateKeyPositions()
          renderKeys()
        }
        spanBtns.appendChild(btn)
      })
      keyDiv.appendChild(spanBtns)
    }

    container.appendChild(keyDiv)
  })

  // 添加按键按钮
  const addBtn = document.createElement('button')
  addBtn.className = 'kv-add-btn'
  addBtn.textContent = '+ 添加按键'
  addBtn.onclick = () => {
    showModal.value = true
    renderModal()
  }
  container.appendChild(addBtn)
}

// 渲染行配置区
const renderRows = () => {
  const container = document.getElementById('rows-container')
  if (!container) return
  container.innerHTML = ''

  rows.forEach(row => {
    const rowDiv = document.createElement('div')
    rowDiv.className = 'kv-config-row'

    const label = document.createElement('span')
    label.className = 'kv-row-label'
    label.style.color = row.color
    label.textContent = `第${row.id + 1}行`
    rowDiv.appendChild(label)

    // 宽度滑块
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
    rowDiv.appendChild(widthItem)

    // 颜色选择器
    const colorItem = document.createElement('div')
    colorItem.className = 'kv-row-config-item'
    colorItem.innerHTML = '<span>色</span>'
    const colorInput = document.createElement('input')
    colorInput.type = 'color'
    colorInput.value = row.color
    colorInput.onchange = () => {
      row.color = colorInput.value
      saveConfig()
      renderRows()
    }
    colorItem.appendChild(colorInput)
    rowDiv.appendChild(colorItem)

    container.appendChild(rowDiv)
  })
}

// 渲染模态框
const renderModal = () => {
  const root = document.getElementById('modal-root')
  if (!root) return

  if (showModal.value) {
    root.innerHTML = `
      <div class="kv-modal-overlay" style="position:fixed;top:0;left:0;right:0;bottom:0;background:rgba(0,0,0,0.6);display:flex;align-items:center;justify-content:center;z-index:9999;" onclick="this.parentElement.innerHTML=''">
        <div class="kv-modal" style="background:var(--vp-c-bg-elv);border:1px solid var(--vp-c-divider);border-radius:12px;padding:20px;width:300px;max-width:90vw;" onclick="event.stopPropagation()">
          <h3 style="margin:0 0 16px 0;color:var(--vp-c-text-1);font-size:16px;">添加按键</h3>
          <input id="modal-input" type="text" placeholder="按键名（如 L）" style="width:100%;padding:8px 12px;border:1px solid var(--vp-c-divider);border-radius:6px;background:var(--vp-c-bg-soft);color:var(--vp-c-text-1);font-size:14px;margin-bottom:16px;">
          <div style="display:flex;gap:8px;margin-bottom:20px;">
            <button class="modal-type" data-type="normal" style="flex:1;padding:8px;border:1px solid var(--vp-c-divider);border-radius:6px;background:var(--vp-c-brand-dimm);color:var(--vp-c-brand);font-size:12px;cursor:pointer;">普通键</button>
            <button class="modal-type" data-type="kps" style="flex:1;padding:8px;border:1px solid var(--vp-c-divider);border-radius:6px;background:var(--vp-c-bg-soft);color:var(--vp-c-text-2);font-size:12px;cursor:pointer;">KPS键</button>
            <button class="modal-type" data-type="total" style="flex:1;padding:8px;border:1px solid var(--vp-c-divider);border-radius:6px;background:var(--vp-c-bg-soft);color:var(--vp-c-text-2);font-size:12px;cursor:pointer;">TOTAL键</button>
          </div>
          <div style="display:flex;gap:12px;">
            <button class="modal-cancel" style="flex:1;padding:10px;border:none;border-radius:6px;background:var(--vp-c-bg-mute);color:var(--vp-c-text-1);cursor:pointer;font-weight:bold;">取消</button>
            <button class="modal-confirm" style="flex:1;padding:10px;border:none;border-radius:6px;background:var(--vp-c-brand);color:white;cursor:pointer;font-weight:bold;">确定</button>
          </div>
        </div>
      </div>
    `
    // 输入框聚焦
    const input = document.getElementById('modal-input') as HTMLInputElement
    input?.focus()

    // 类型选择
    root.querySelectorAll('.modal-type').forEach(btn => {
      btn.addEventListener('click', (e) => {
        const target = e.target as HTMLElement
        newKeyType.value = target.dataset.type as any
        root.querySelectorAll('.modal-type').forEach(b => {
          (b as HTMLElement).style.borderColor = 'var(--vp-c-divider)'
          ;(b as HTMLElement).style.background = 'var(--vp-c-bg-soft)'
          ;(b as HTMLElement).style.color = 'var(--vp-c-text-2)'
        })
        target.style.borderColor = 'var(--vp-c-brand)'
        target.style.background = 'var(--vp-c-brand-dimm)'
        target.style.color = 'var(--vp-c-brand)'
      })
    })

    // 取消按钮
    root.querySelector('.modal-cancel')?.addEventListener('click', () => {
      showModal.value = false
      renderModal()
    })

    // 确定按钮
    root.querySelector('.modal-confirm')?.addEventListener('click', () => {
      const label = (document.getElementById('modal-input') as HTMLInputElement).value.trim() || 'N'
      const code = newKeyType.value === 'normal' ? `Key${label.toUpperCase()}` : `INFO_${nextKeyId.value}`
      keys.push({
        id: nextKeyId.value++,
        label,
        code,
        rowId: 0,
        cnt: 0,
        type: newKeyType.value,
        span: 1,
        x: 0, y: 0, w: 0, h: 0
      })
      newKeyLabel.value = ''
      newKeyType.value = 'normal'
      showModal.value = false
      saveConfig()
      updateKeyPositions()
      renderKeys()
      renderModal()
    })

    // 回车确认
    input?.addEventListener('keydown', (e) => {
      if (e.key === 'Enter') {
        root.querySelector('.modal-confirm')?.dispatchEvent(new Event('click'))
      }
    })
  } else {
    root.innerHTML = ''
  }
}

onMounted(() => {
  // 初始化Canvas
  canvas = document.getElementById('kv-canvas') as HTMLCanvasElement
  ctx = canvas.getContext('2d')
  containerRect = canvas.parentElement!.getBoundingClientRect()
  dpr = window.devicePixelRatio || 1

  // 加载配置
  loadConfig()
  updateThemeColors()
  
  const initCanvas = () => {
    if (!canvas || !containerRect) return
    canvas.width = containerRect.width * dpr
    canvas.height = 320 * dpr
    canvas.style.width = `${containerRect.width}px`
    canvas.style.height = '320px'
    ctx?.scale(dpr, dpr)
    updateKeyPositions()
    renderKeys()
    renderRows()
  }
  initCanvas()

  // 重置按钮
  document.getElementById('reset-btn')?.addEventListener('click', resetCounts)

  // 主题监听
  const observer = new MutationObserver(updateThemeColors)
  observer.observe(document.documentElement, { attributes: true, attributeFilter: ['class'] })

  // 窗口 resize
  window.addEventListener('resize', () => {
    if (!canvas) return
    containerRect = canvas.parentElement!.getBoundingClientRect()
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
    document.getElementById('kps-num')!.textContent = kps.value.toString()
    document.getElementById('total-num')!.textContent = total.value.toString()
    requestAnimationFrame(animate)
  }
  animate()

  // 数据变化时重绘配置
  watch([keys, rows], () => {
    renderKeys()
    renderRows()
  }, { deep: true })

  watch(showModal, renderModal)

  onUnmounted(() => {
    observer.disconnect()
    document.removeEventListener('keydown', keydownHandler)
    document.removeEventListener('keyup', keyupHandler)
  })
})
</script>

<style>
/* 样式和你之前的一模一样，没改 */
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
</style>

## 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。
