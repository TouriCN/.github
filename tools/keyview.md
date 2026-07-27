# KeyView
<div class="kv-wrapper">
  <canvas id="kv-canvas" class="kv-canvas"></canvas>
  
  <div class="kv-info-bar">
    <div class="kv-kps">KPS: <span id="kps-num">0</span></div>
    <div class="kv-total">TOTAL: <span id="total-num">0</span></div>
  </div>

  <div class="kv-config">
    <div class="kv-config-keys" id="keys-container"></div>
    <div class="kv-config-rows" id="rows-container"></div>
  </div>

  <div id="modal-root"></div>
</div>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, watch } from 'vue'

const getCssVar = (name: string): string => {
  return getComputedStyle(document.documentElement).getPropertyValue(name).trim()
}

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

// 缓存DOM引用，避免反复querySelector
let keysContainer: HTMLElement | null = null
let rowsContainer: HTMLElement | null = null

const updateThemeColors = () => {
  themeColors.bgSoft = getCssVar('--vp-c-bg-soft') || '#f6f6f7'
  themeColors.divider = getCssVar('--vp-c-divider') || '#e2e8f0'
  themeColors.text1 = getCssVar('--vp-c-text-1') || '#1a202c'
  themeColors.text3 = getCssVar('--vp-c-text-3') || '#64748b'
}

const saveConfig = () => {
  try {
    localStorage.setItem('keyview_config', JSON.stringify({
      rows: rows.map(r => ({...r})),
      keys: keys.map(k => ({...k, x: 0, y: 0, w: 0, h: 0})),
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

// ========== 核心修复：配置区DOM只构建一次，之后只更新active类 ==========
// 首次完整渲染按键配置
const renderKeysInitial = () => {
  if (!keysContainer) return
  keysContainer.innerHTML = ''

  keys.forEach(key => {
    const keyDiv = document.createElement('div')
    keyDiv.className = 'kv-config-key'
    keyDiv.dataset.keyId = key.id.toString()

    if (key.type === 'normal') {
      const delBtn = document.createElement('button')
      delBtn.className = 'kv-del-btn'
      delBtn.textContent = '×'
      delBtn.onclick = () => {
        const idx = keys.findIndex(k => k.id === key.id)
        if (idx > -1) {
          keys.splice(idx, 1)
          saveConfig()
          renderKeysInitial() // 删除操作才重建
          updateKeyPositions()
        }
      }
      keyDiv.appendChild(delBtn)
    }

    const label = document.createElement('span')
    label.className = 'kv-key-label'
    label.textContent = key.label
    keyDiv.appendChild(label)

    const rowBtns = document.createElement('div')
    rowBtns.className = 'kv-row-btns'
    rows.forEach(row => {
      const btn = document.createElement('button')
      btn.className = `kv-row-btn ${key.rowId === row.id ? 'active' : ''}`
      btn.dataset.rowId = row.id.toString()
      btn.textContent = (row.id + 1).toString()
      btn.onclick = () => {
        key.rowId = row.id
        saveConfig()
        // 只更新active类，不重建DOM
        rowBtns.querySelectorAll('.kv-row-btn').forEach(b => {
          const isActive = b.dataset.rowId === row.id.toString()
          b.classList.toggle('active', isActive)
        })
      }
      rowBtns.appendChild(btn)
    })
    keyDiv.appendChild(rowBtns)

    if (key.type === 'normal') {
      const spanBtns = document.createElement('div')
      spanBtns.className = 'kv-span-btns'
      ;[1,2,3,4].forEach(span => {
        const btn = document.createElement('button')
        btn.className = `kv-span-btn ${key.span === span ? 'active' : ''}`
        btn.dataset.span = span.toString()
        btn.textContent = span.toString()
        btn.onclick = () => {
          key.span = span
          saveConfig()
          updateKeyPositions()
          // 只更新active类，不重建DOM
          spanBtns.querySelectorAll('.kv-span-btn').forEach(b => {
            const isActive = b.dataset.span === span.toString()
            b.classList.toggle('active', isActive)
          })
        }
        spanBtns.appendChild(btn)
      })
      keyDiv.appendChild(spanBtns)
    }

    keysContainer.appendChild(keyDiv)
  })

  const addBtn = document.createElement('button')
  addBtn.className = 'kv-add-btn'
  addBtn.textContent = '+ 添加按键'
  addBtn.onclick = () => {
    showModal.value = true
    renderModal()
  }
  keysContainer.appendChild(addBtn)
}

// 新增按键时，只追加一个新节点，不重建整个列表
const appendNewKey = (key: any) => {
  if (!keysContainer) return
  // 找到"添加按键"按钮，在新键后面插入
  const addBtn = keysContainer.querySelector('.kv-add-btn')
  const keyDiv = document.createElement('div')
  keyDiv.className = 'kv-config-key'
  keyDiv.dataset.keyId = key.id.toString()

  const label = document.createElement('span')
  label.className = 'kv-key-label'
  label.textContent = key.label
  keyDiv.appendChild(label)

  const rowBtns = document.createElement('div')
  rowBtns.className = 'kv-row-btns'
  rows.forEach(row => {
    const btn = document.createElement('button')
    btn.className = `kv-row-btn ${key.rowId === row.id ? 'active' : ''}`
    btn.dataset.rowId = row.id.toString()
    btn.textContent = (row.id + 1).toString()
    btn.onclick = () => {
      key.rowId = row.id
      saveConfig()
      rowBtns.querySelectorAll('.kv-row-btn').forEach(b => {
        b.classList.toggle('active', b.dataset.rowId === row.id.toString())
      })
    }
    rowBtns.appendChild(btn)
  })
  keyDiv.appendChild(rowBtns)

  const spanBtns = document.createElement('div')
  spanBtns.className = 'kv-span-btns'
  ;[1,2,3,4].forEach(span => {
    const btn = document.createElement('button')
    btn.className = `kv-span-btn ${key.span === span ? 'active' : ''}`
    btn.dataset.span = span.toString()
    btn.textContent = span.toString()
    btn.onclick = () => {
      key.span = span
      saveConfig()
      updateKeyPositions()
      spanBtns.querySelectorAll('.kv-span-btn').forEach(b => {
        b.classList.toggle('active', b.dataset.span === span.toString())
      })
    }
    spanBtns.appendChild(btn)
  })
  keyDiv.appendChild(spanBtns)

  if (addBtn) {
    keysContainer.insertBefore(keyDiv, addBtn)
  } else {
    keysContainer.appendChild(keyDiv)
  }
}

// 首次完整渲染行配置
const renderRowsInitial = () => {
  if (!rowsContainer) return
  rowsContainer.innerHTML = ''

  rows.forEach(row => {
    const rowDiv = document.createElement('div')
    rowDiv.className = 'kv-config-row'
    rowDiv.dataset.rowId = row.id.toString()

    const label = document.createElement('span')
    label.className = 'kv-row-label'
    label.style.color = row.color
    label.textContent = `第${row.id + 1}行`
    rowDiv.appendChild(label)

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

    const colorItem = document.createElement('div')
    colorItem.className = 'kv-row-config-item'
    colorItem.innerHTML = '<span>色</span>'
    const colorInput = document.createElement('input')
    colorInput.type = 'color'
    colorInput.value = row.color
    colorInput.onchange = () => {
      row.color = colorInput.value
      saveConfig()
      // 只更新标签颜色，不重建
      label.style.color = row.color
    }
    colorItem.appendChild(colorInput)
    rowDiv.appendChild(colorItem)

    rowsContainer.appendChild(rowDiv)
  })
}

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
    const input = document.getElementById('modal-input') as HTMLInputElement
    input?.focus()

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

    root.querySelector('.modal-cancel')?.addEventListener('click', () => {
      showModal.value = false
      renderModal()
    })

    root.querySelector('.modal-confirm')?.addEventListener('click', () => {
      const label = (document.getElementById('modal-input') as HTMLInputElement).value.trim() || 'N'
      const code = newKeyType.value === 'normal' ? `Key${label.toUpperCase()}` : `INFO_${nextKeyId.value}`
      const newKey = {
        id: nextKeyId.value++,
        label,
        code,
        rowId: 0,
        cnt: 0,
        type: newKeyType.value,
        span: 1,
        x: 0, y: 0, w: 0, h: 0
      }
      keys.push(newKey)
      newKeyLabel.value = ''
      newKeyType.value = 'normal'
      showModal.value = false
      saveConfig()
      updateKeyPositions()
      appendNewKey(newKey) // 只追加新节点
      renderModal()
    })

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
  canvas = document.getElementById('kv-canvas') as HTMLCanvasElement
  keysContainer = document.getElementById('keys-container')
  rowsContainer = document.getElementById('rows-container')
  ctx = canvas.getContext('2d')
  containerRect = canvas.parentElement!.getBoundingClientRect()
  dpr = window.devicePixelRatio || 1

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
    renderKeysInitial()  // 只构建一次
    renderRowsInitial()  // 只构建一次
  }
  initCanvas()

  // 点击Canvas区域时主动获取焦点，确保能接收键盘事件
  canvas.addEventListener('click', () => {
    window.focus()
  })

  const observer = new MutationObserver(updateThemeColors)
  observer.observe(document.documentElement, { attributes: true, attributeFilter: ['class'] })

  window.addEventListener('resize', () => {
    if (!canvas) return
    containerRect = canvas.parentElement!.getBoundingClientRect()
    initCanvas()
  })

  // 键盘事件：加isComposing判断，避免输入法冲突
  const keydownHandler = (e: KeyboardEvent) => {
    // 输入法组合中（中文拼音上屏等），不触发游戏键
    if (e.isComposing) return
    // 如果焦点在输入框/文本域，不拦截（避免影响其他输入）
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName || '')) return
    const key = keys.find(k => k.code === e.code && k.type === 'normal')
    if (key) {
      e.preventDefault()  // 阻止默认行为（如F键打开搜索等）
      handleDown(e.code)
    }
  }
  const keyupHandler = (e: KeyboardEvent) => {
    if (e.isComposing) return
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName || '')) return
    const key = keys.find(k => k.code === e.code && k.type === 'normal')
    if (key) handleUp(e.code)
  }
  document.addEventListener('keydown', keydownHandler)
  document.addEventListener('keyup', keyupHandler)
  window.addEventListener('blur', () => pressed.clear())

  const animate = () => {
    if (!ctx || !containerRect) return
    ctx.clearRect(0, 0, containerRect.width, 320)
    drawBars()
    drawKeys()
    const now = Date.now()
    while (kpsRecords.length && kpsRecords[0] < now - 1000) kpsRecords.shift()
    kps.value = kpsRecords.length
    document.getElementById('kps-num')!.textContent = kps.value.toString()
    document.getElementById('total-num')!.textContent = total.value.toString()
    requestAnimationFrame(animate)
  }
  animate()

  // 【已移除】之前的watch([keys, rows])深监听，那个是导致DOM反复重建的元凶
  watch(showModal, renderModal)

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
  /* 确保Canvas可点击获取焦点上下文 */
  outline: none;
}

.kv-info-bar {
  position: absolute;
  bottom: 210px;
  left: 10px;
  right: 10px;
  display: flex;
  gap: 12px;
  align-items: center;
  pointer-events: none; /* 信息栏不拦截点击 */
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
