# KeyView

<div id="kv-root"></div>

<script setup lang="ts">
import { onMounted } from 'vue'

onMounted(() => {
  // 所有代码都在这里运行，构建时不会碰，运行时才执行
  const root = document.getElementById('kv-root')
  if (!root) return

  // 注入HTML结构
  root.innerHTML = `
    <div id="kv-container">
      <div id="kv-main"></div>
      <div id="kv-keys"></div>
      <div id="kv-cfg">
        <div id="kv-cfg-top"></div>
        <div id="kv-cfg-bottom"></div>
      </div>
    </div>
  `

  // ===== 配置数据 =====
  const ROWS = [
    { id: 0, width: 52, color: '#ff3366' },
    { id: 1, width: 40, color: '#ff9f43' },
    { id: 2, width: 28, color: '#ffcc00' },
    { id: 3, width: 16, color: '#00ff88' }
  ]

  let keys = [
    { id: 1, label: 'F', code: 'KeyF', rowId: 0, cnt: 0, type: 'normal', span: 1 },
    { id: 2, label: 'G', code: 'KeyG', rowId: 0, cnt: 0, type: 'normal', span: 1 },
    { id: 3, label: 'H', code: 'KeyH', rowId: 0, cnt: 0, type: 'normal', span: 1 },
    { id: 4, label: 'J', code: 'KeyJ', rowId: 0, cnt: 0, type: 'normal', span: 1 },
    { id: 5, label: 'KPS', code: 'KPS_INFO', rowId: 1, cnt: 0, type: 'kps', span: 2 },
    { id: 6, label: 'TOTAL', code: 'TOTAL_INFO', rowId: 1, cnt: 0, type: 'total', span: 2 }
  ]

  let nextKeyId = 7
  let kps = 0
  let total = 0
  let pressedKeys = new Set<string>()
  let bars: any[] = []
  let barId = 0
  let kpsRecords: number[] = []

  // ===== 样式注入（避免外部CSS冲突）=====
  const style = document.createElement('style')
  style.textContent = `
    #kv-container { width:100%; height:520px; background: var(--vp-c-bg); border:1px solid var(--vp-c-divider); border-radius:8px; position:relative; overflow:hidden; margin:24px 0; }
    #kv-container #kv-main { position:absolute; inset:0; bottom:200px; z-index:2; }
    #kv-container #kv-keys { position:absolute; left:0; right:0; bottom:200px; display:flex; flex-direction:column; align-items:center; gap:14px; padding:10px 0; pointer-events:none; z-index:1; }
    #kv-container .kv-row { display:flex; gap:12px; pointer-events:auto; }
    #kv-container .key { min-width:50px; height:50px; border:2px solid var(--vp-c-divider); border-radius:8px; font-weight:bold; display:flex; flex-direction:column; align-items:center; justify-content:center; font-size:15px; background: var(--vp-c-bg-soft); color: var(--vp-c-text-1); cursor:pointer; transition: transform 0.05s ease, border-color 0.05s ease; will-change: transform; }
    #kv-container .key.pressed { transform: scale(0.96); border-color: var(--vp-c-brand); background: var(--vp-c-brand-dimm); }
    #kv-container .info-key { background: var(--vp-c-bg-mute) !important; cursor:default; min-width:110px; }
    #kv-container .kps-key .kv-val { color:#ff9f43; font-size:22px; }
    #kv-container .total-key .kv-val { color:#00b96b; font-size:22px; }
    #kv-container #kv-cfg { position:absolute; bottom:0; left:0; right:0; height:200px; background: var(--vp-c-bg-soft); border-top:1px solid var(--vp-c-divider); display:flex; flex-direction:column; }
    #kv-container .kv-bar { position:absolute; border-radius:2px; will-change: transform, opacity; animation: barRise 3s linear forwards; z-index:10; }
    @keyframes barRise {
      0% { height:2px; opacity:1; bottom:2px; }
      33% { height: var(--bar-height); opacity:1; }
      66% { height: var(--bar-height); opacity:1; transform: translateY(calc(-1 * var(--bar-height))); }
      100% { height:2px; opacity:0; transform: translateY(calc(-1 * var(--bar-height) * 2)); }
    }
  `
  document.head.appendChild(style)

  // ===== 渲染按键 =====
  const renderKeys = () => {
    const keysEl = document.getElementById('kv-keys')
    if (!keysEl) return
    keysEl.innerHTML = ''

    const rowMap: Record<number, typeof keys> = {}
    keys.forEach(k => {
      if (!rowMap[k.rowId]) rowMap[k.rowId] = []
      rowMap[k.rowId].push(k)
    })

    Object.values(rowMap).forEach(rowKeys => {
      const rowEl = document.createElement('div')
      rowEl.className = 'kv-row'
      rowKeys.forEach(key => {
        const keyEl = document.createElement('div')
        keyEl.className = 'key'
        keyEl.dataset.code = key.code
        keyEl.style.width = `${50 + (key.span - 1) * 56}px`
        
        if (key.type === 'kps') {
          keyEl.innerHTML = `<span class="kv-lbl">KPS</span><span class="kv-val">${kps}</span>`
          keyEl.classList.add('info-key', 'kps-key')
        } else if (key.type === 'total') {
          keyEl.innerHTML = `<span class="kv-lbl">TOTAL</span><span class="kv-val">${total}</span>`
          keyEl.classList.add('info-key', 'total-key')
        } else {
          keyEl.textContent = key.label
          const cnt = document.createElement('span')
          cnt.className = 'kv-cnt'
          cnt.textContent = String(key.cnt)
          keyEl.appendChild(cnt)
        }

        keyEl.addEventListener('mousedown', () => handleKeyDown(key.code))
        keyEl.addEventListener('mouseup', () => handleKeyUp(key.code))
        keyEl.addEventListener('touchstart', (e) => { e.preventDefault(); handleKeyDown(key.code) })
        keyEl.addEventListener('touchend', (e) => { e.preventDefault(); handleKeyUp(key.code) })
        
        rowEl.appendChild(keyEl)
      })
      keysEl.appendChild(rowEl)
    })
  }

  // ===== 核心逻辑 =====
  const handleKeyDown = (code: string) => {
    if (pressedKeys.has(code)) return
    pressedKeys.add(code)

    keys.forEach(key => {
      if (key.code === code && key.type === 'normal') {
        key.cnt++
        total++
        const keyEl = document.querySelector(`#kv-container .key[data-code="${code}"]`)
        const mainEl = document.getElementById('kv-main')
        if (keyEl && mainEl) {
          const keyRect = keyEl.getBoundingClientRect()
          const mainRect = mainEl.getBoundingClientRect()
          const row = ROWS[key.rowId]
          
          const bar = document.createElement('div')
          bar.className = 'kv-bar'
          bar.style.cssText = `
            --bar-height: ${row.width}px;
            width: ${row.width}px;
            height: 2px;
            background: ${row.color};
            left: ${keyRect.left - mainRect.left + (keyRect.width - row.width) / 2}px;
            bottom: 2px;
          `
          mainEl.appendChild(bar)
          setTimeout(() => bar.remove(), 3000)
        }
      }
    })

    kpsRecords.push(Date.now())
    renderKeys() // 更新KPS/TOTAL显示
  }

  const handleKeyUp = (code: string) => {
    pressedKeys.delete(code)
  }

  // ===== 键盘监听（最原始的方式，100%有效）=====
  document.addEventListener('keydown', (e) => {
    const isInput = ['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName || '')
    if (!isInput) {
      e.preventDefault()
      handleKeyDown(e.code)
    }
  })

  document.addEventListener('keyup', (e) => {
    const isInput = ['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName || '')
    if (!isInput) {
      e.preventDefault()
      handleKeyUp(e.code)
    }
  })

  // ===== 初始化 =====
  const updateKPS = () => {
    const now = Date.now()
    kpsRecords = kpsRecords.filter(t => now - t < 1000)
    kps = kpsRecords.length
    renderKeys()
    requestAnimationFrame(updateKPS)
  }

  renderKeys()
  updateKPS()
})
</script>

## 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。
