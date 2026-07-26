# KeyView

<div id="kv-container">
  <canvas id="kv-canvas"></canvas>
  <!-- 只有静态交互层用DOM，比如弹窗、配置按钮，不参与动态渲染 -->
  <div id="kv-interaction">
    <div id="kv-cfg">
      <div id="kv-cfg-top"></div>
      <div id="kv-cfg-bottom"></div>
    </div>
    <div class="kv-modal-overlay" id="kv-modal">
      <div class="kv-modal-box">
        <h3>添加按键</h3>
        <input type="text" id="kv-newKeyName" placeholder="输入按键名（如 L）">
        <div class="kv-select-group">
          <label>按键类型</label>
          <div class="kv-type-btns">
            <div class="kv-type-btn normal on" data-type="normal">普通键</div>
            <div class="kv-type-btn kps" data-type="kps">KPS键</div>
            <div class="kv-type-btn total" data-type="total">TOTAL键</div>
          </div>
        </div>
        <div class="kv-select-group">
          <label>绑定行</label>
          <div class="kv-row-btns" id="kv-rowSelectBtns"></div>
        </div>
        <div class="kv-actions">
          <button class="kv-btn cancel" id="kv-modalCancel">取消</button>
          <button class="kv-btn ok" id="kv-modalOk">确定</button>
        </div>
      </div>
    </div>
  </div>
</div>

<style>
#kv-container { width:100%; height:520px; background:var(--vp-c-bg); border:1px solid var(--vp-c-divider); border-radius:8px; position:relative; overflow:hidden; margin:24px 0; }
#kv-canvas { position:absolute; inset:0; z-index:1; } /* Canvas铺满容器，画所有动态内容 */
#kv-interaction { position:absolute; inset:0; z-index:2; pointer-events:none; } /* 交互层只接收点击，不挡Canvas */
#kv-interaction * { pointer-events:auto; } /* 交互元素恢复点击 */
#kv-container #kv-cfg { position:absolute; bottom:0; left:0; right:0; height:200px; background:var(--vp-c-bg-soft); border-top:1px solid var(--vp-c-divider); display:flex; flex-direction:column; pointer-events:auto; }
#kv-container #kv-cfg-top { flex:1; display:flex; gap:12px; padding:10px; overflow:auto; }
#kv-container #kv-cfg-bottom { height:80px; display:flex; gap:16px; padding:0 12px; align-items:center; overflow:auto; }
#kv-container .cfg-item { min-width:72px; padding:6px; border:1px solid var(--vp-c-divider); border-radius:6px; background:var(--vp-c-bg-mute); display:flex; flex-direction:column; align-items:center; gap:3px; position:relative; }
#kv-container .del-btn { position:absolute; top:-5px; right:-5px; width:15px; height:15px; border-radius:50%; background:var(--vp-c-danger); color:white; border:none; cursor:pointer; font-size:10px; }
#kv-container .row-btn { width:16px; height:16px; border-radius:2px; border:1px solid var(--vp-c-divider); background:var(--vp-c-bg-soft); cursor:pointer; font-size:8px; }
#kv-container .row-btn.on { border-color:var(--vp-c-brand); background:var(--vp-c-brand-dimm); }
#kv-container .span-btn { width:14px; height:14px; border-radius:2px; border:1px solid var(--vp-c-divider); background:var(--vp-c-bg-soft); cursor:pointer; font-size:7px; }
#kv-container .span-btn.on { border-color:var(--vp-c-brand); background:var(--vp-c-brand-dimm); }
#kv-container .add-btn { padding:8px 14px; border:2px solid var(--vp-c-brand); background:var(--vp-c-bg-soft); color:var(--vp-c-brand); border-radius:6px; cursor:pointer; }
#kv-container .kv-modal-overlay { display:none; position:fixed; inset:0; background:rgba(0,0,0,0.6); z-index:99999; align-items:center; justify-content:center; }
#kv-container .kv-modal-overlay.show { display:flex; }
#kv-container .kv-modal-box { background:var(--vp-c-bg-elv); border:1px solid var(--vp-c-divider); border-radius:12px; padding:20px; width:320px; }
#kv-container .kv-type-btn { flex:1; padding:8px; border:2px solid var(--vp-c-divider); border-radius:4px; cursor:pointer; font-size:11px; text-align:center; }
#kv-container .kv-type-btn.on { border-color:var(--vp-c-brand); background:var(--vp-c-brand-dimm); }
#kv-container .kv-btn { flex:1; padding:10px; border:none; border-radius:6px; cursor:pointer; font-weight:bold; }
#kv-container .kv-btn.cancel { background:var(--vp-c-bg-mute); }
#kv-container .kv-btn.ok { background:var(--vp-c-brand); color:white; }
</style>

<script setup lang="ts">
import { onMounted, ref, onUnmounted } from 'vue'

// 内联Web Worker：所有计算逻辑放这里，完全不阻塞主线程
const kpsWorker = new Worker(new URL('./kps.worker.ts', import.meta.url), { type: 'module' })

onMounted(() => {
  const container = document.getElementById('kv-container') as HTMLDivElement
  const canvas = document.getElementById('kv-canvas') as HTMLCanvasElement
  const ctx = canvas.getContext('2d')!
  const interaction = document.getElementById('kv-interaction')!

  // 初始化Canvas尺寸（适配高清屏，避免模糊）
  const dpr = window.devicePixelRatio || 1
  canvas.width = container.clientWidth * dpr
  canvas.height = container.clientHeight * dpr
  ctx.scale(dpr, dpr)
  canvas.style.width = `${container.clientWidth}px`
  canvas.style.height = `${container.clientHeight}px`

  // ===== 配置数据（和之前一致，方便迁移）=====
  const ROWS = [
    { id: 0, width: 52, color: '#ff3366' },
    { id: 1, width: 40, color: '#ff9f43' },
    { id: 2, width: 28, color: '#ffcc00' },
    { id: 3, width: 16, color: '#00ff88' }
  ]
  let keys = [
    { id: 1, label: 'F', code: 'KeyF', rowId: 0, cnt: 0, type: 'normal', span: 1, x: 0, y: 0, width: 50, height: 50 },
    { id: 2, label: 'G', code: 'KeyG', rowId: 0, cnt: 0, type: 'normal', span: 1, x: 0, y: 0, width: 50, height: 50 },
    { id: 3, label: 'H', code: 'KeyH', rowId: 0, cnt: 0, type: 'normal', span: 1, x: 0, y: 0, width: 50, height: 50 },
    { id: 4, label: 'J', code: 'KeyJ', rowId: 0, cnt: 0, type: 'normal', span: 1, x: 0, y: 0, width: 50, height: 50 },
    { id: 5, label: 'KPS', code: 'KPS_INFO', rowId: 1, cnt: 0, type: 'kps', span: 2, x: 0, y: 0, width: 110, height: 50 },
    { id: 6, label: 'TOTAL', code: 'TOTAL_INFO', rowId: 1, cnt: 0, type: 'total', span: 2, x: 0, y: 0, width: 110, height: 50 }
  ]
  let nextKeyId = 7
  let kps = 0
  let total = 0
  let pressedKeys = new Set<string>()
  let bars: Array<{ x: number, y: number, height: number, color: string, alpha: number }> = []
  let frameCount = 0
  let lastFpsUpdate = performance.now()
  let currentFps = 60
  let simplifyAnimation = false // 帧率低时简化动画

  // ===== 双缓冲渲染（避免闪烁）=====
  const bufferCanvas = document.createElement('canvas')
  const bufferCtx = bufferCanvas.getContext('2d')!
  bufferCanvas.width = canvas.width
  bufferCanvas.height = canvas.height

  // ===== 计算按键位置（只算一次，除非窗口 resize）=====
  const calculateKeyPositions = () => {
    const containerWidth = container.clientWidth
    const keyAreaTop = container.clientHeight - 200 - 70 // 减去配置栏高度和间距
    const rowSpacing = 14
    const keySpacing = 12

    // 按行分组
    const rowMap: Record<number, typeof keys> = {}
    keys.forEach(key => {
      if (!rowMap[key.rowId]) rowMap[key.rowId] = []
      rowMap[key.rowId].push(key)
    })

    // 计算每个按键的位置
    Object.values(rowMap).forEach((rowKeys, rowIndex) => {
      const totalRowWidth = rowKeys.reduce((sum, key) => sum + (50 + (key.span - 1) * 56), 0) + (rowKeys.length - 1) * keySpacing
      let currentX = (containerWidth - totalRowWidth) / 2
      const currentY = keyAreaTop + rowIndex * (50 + rowSpacing)

      rowKeys.forEach(key => {
        key.x = currentX
        key.y = currentY
        key.width = 50 + (key.span - 1) * 56
        key.height = 50
        currentX += key.width + keySpacing
      })
    })
  }

  // ===== Canvas绘制逻辑（核心性能点）=====
  const draw = () => {
    const start = performance.now()
    frameCount++

    // 清空缓冲区
    bufferCtx.clearRect(0, 0, bufferCanvas.width, bufferCanvas.height)

    // 1. 画按键
    keys.forEach(key => {
      // 按键背景
      bufferCtx.fillStyle = pressedKeys.has(key.code) ? 'var(--vp-c-brand-dimm)' : 'var(--vp-c-bg-soft)'
      bufferCtx.strokeStyle = pressedKeys.has(key.code) ? 'var(--vp-c-brand)' : 'var(--vp-c-divider)'
      bufferCtx.lineWidth = 2
      bufferCtx.beginPath()
      bufferCtx.roundRect(key.x, key.y, key.width, key.height, 8)
      bufferCtx.fill()
      bufferCtx.stroke()

      // 按键文字
      bufferCtx.fillStyle = 'var(--vp-c-text-1)'
      bufferCtx.font = '15px sans-serif'
      bufferCtx.textAlign = 'center'
      bufferCtx.textBaseline = 'middle'
      bufferCtx.fillText(key.label, key.x + key.width / 2, key.y + key.height / 2)

      // 按键计数（普通键）
      if (key.type === 'normal') {
        bufferCtx.font = '10px sans-serif'
        bufferCtx.fillStyle = 'var(--vp-c-text-3)'
        bufferCtx.fillText(`${key.cnt}`, key.x + key.width / 2, key.y + key.height / 2 + 12)
      }

      // KPS/TOTAL数值
      if (key.type === 'kps') {
        bufferCtx.font = '22px sans-serif'
        bufferCtx.fillStyle = '#ff9f43'
        bufferCtx.fillText(`${kps}`, key.x + key.width / 2, key.y + key.height / 2)
        bufferCtx.font = '9px sans-serif'
        bufferCtx.fillStyle = 'var(--vp-c-text-3)'
        bufferCtx.fillText('KPS', key.x + key.width / 2, key.y + 10)
      }
      if (key.type === 'total') {
        bufferCtx.font = '22px sans-serif'
        bufferCtx.fillStyle = '#00b96b'
        bufferCtx.fillText(`${total}`, key.x + key.width / 2, key.y + key.height / 2)
        bufferCtx.font = '9px sans-serif'
        bufferCtx.fillStyle = 'var(--vp-c-text-3)'
        bufferCtx.fillText('TOTAL', key.x + key.width / 2, key.y + 10)
      }
    })

    // 2. 画上升条（简化模式下只画线，不画渐变）
    bars.forEach(bar => {
      if (simplifyAnimation) {
        bufferCtx.fillStyle = bar.color
        bufferCtx.globalAlpha = bar.alpha
        bufferCtx.fillRect(bar.x, bar.y, 4, bar.height)
      } else {
        // 复杂模式：渐变效果
        const gradient = bufferCtx.createLinearGradient(bar.x, bar.y, bar.x, bar.y + bar.height)
        gradient.addColorStop(0, bar.color)
        gradient.addColorStop(1, 'transparent')
        bufferCtx.fillStyle = gradient
        bufferCtx.globalAlpha = bar.alpha
        bufferCtx.fillRect(bar.x, bar.y, 6, bar.height)
      }
    })
    bufferCtx.globalAlpha = 1

    // 把缓冲区内容复制到主Canvas
    ctx.clearRect(0, 0, canvas.width, canvas.height)
    ctx.drawImage(bufferCanvas, 0, 0)

    // 帧率检测：连续3帧低于55fps就开简化模式
    if (performance.now() - lastFpsUpdate >= 1000) {
      currentFps = frameCount
      simplifyAnimation = currentFps < 55
      frameCount = 0
      lastFpsUpdate = performance.now()
    }

    requestAnimationFrame(draw)
  }

  // ===== 上升条更新逻辑（丢给Worker的话会增加通信成本，这里轻量计算直接放主线程）=====
  const updateBars = () => {
    bars = bars.filter(bar => {
      bar.y -= 3 // 上升速度
      bar.alpha -= 0.01 // 淡出
      return bar.y > 0 && bar.alpha > 0
    })
  }
  const barInterval = setInterval(updateBars, 16) // 60fps更新

  // ===== 交互逻辑 =====
  // 1. Canvas点击检测（判断点了哪个按键）
  canvas.addEventListener('click', (e) => {
    const rect = canvas.getBoundingClientRect()
    const x = (e.clientX - rect.left) * (canvas.width / rect.width) / dpr
    const y = (e.clientY - rect.top) * (canvas.height / rect.height) / dpr

    const clickedKey = keys.find(key => 
      x >= key.x && x <= key.x + key.width &&
      y >= key.y && y <= key.y + key.height &&
      key.type === 'normal'
    )
    if (clickedKey) {
      handleKeyDown(clickedKey.code)
      setTimeout(() => handleKeyUp(clickedKey.code), 100)
    }
  })

  // 2. 键盘事件（发给Worker统计KPS）
  const handleKeyDown = (code: string) => {
    if (pressedKeys.has(code)) return
    pressedKeys.add(code)
    kpsWorker.postMessage('press') // 通知Worker记录按键

    // 普通键计数
    keys.forEach(key => {
      if (key.code === code && key.type === 'normal') {
        key.cnt++
        total++
        // 加上升条
        const row = ROWS[key.rowId]
        bars.push({
          x: key.x + key.width / 2 - row.width / 2,
          y: key.y,
          height: 2,
          color: row.color,
          alpha: 1
        })
      }
    })
  }
  const handleKeyUp = (code: string) => {
    pressedKeys.delete(code)
  }

  window.addEventListener('keydown', (e) => {
    if (document.getElementById('kv-modal')!.classList.contains('show')) {
      if (e.key === 'Escape') document.getElementById('kv-modal')!.classList.remove('show')
      return
    }
    const isInput = ['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName)
    if (!isInput) e.preventDefault()
    handleKeyDown(e.code)
  })
  window.addEventListener('keyup', (e) => {
    if (document.getElementById('kv-modal')!.classList.contains('show')) return
    const isInput = ['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName)
    if (!isInput) e.preventDefault()
    handleKeyUp(e.code)
  })

  // 3. 配置栏交互（和之前逻辑一致，只操作DOM，不参与动态渲染）
  const renderCfg = () => {
    const top = document.getElementById('kv-cfg-top')!
    top.innerHTML = ''
    keys.forEach(key => {
      const item = document.createElement('div')
      item.className = 'cfg-item'
      item.innerHTML = `
        <button class="del-btn" data-id="${key.id}">×</button>
        <span class="nm">${key.label}</span>
        <div class="row-btns">${ROWS.map(r => `<div class="row-btn ${key.rowId===r.id?'on':''}" data-id="${key.id}" data-rid="${r.id}">${r.id+1}</div>`).join('')}</div>
        <div class="span-btns">${[1,2,3,4].map(s => `<div class="span-btn ${key.span===s?'on':''}" data-id="${key.id}" data-span="${s}">${s}</div>`).join('')}</div>
      `
      top.appendChild(item)
    })
    // 绑定事件...（和之前一致，省略重复代码）
    const addBtn = document.createElement('div')
    addBtn.className = 'add-btns'
    addBtn.innerHTML = '<button class="add-btn">+ 键</button>'
    addBtn.querySelector('.add-btn')!.addEventListener('click', () => {
      document.getElementById('kv-modal')!.classList.add('show')
    })
    top.appendChild(addBtn)
  }

  // ===== Worker通信 =====
  kpsWorker.onmessage = (e) => {
    kps = e.data
  }

  // ===== 初始化 =====
  calculateKeyPositions()
  renderCfg()
  draw()

  // 窗口 resize 重新计算位置
  window.addEventListener('resize', () => {
    const dpr = window.devicePixelRatio || 1
    canvas.width = container.clientWidth * dpr
    canvas.height = container.clientHeight * dpr
    ctx.scale(dpr, dpr)
    canvas.style.width = `${container.clientWidth}px`
    canvas.style.height = `${container.clientHeight}px`
    bufferCanvas.width = canvas.width
    bufferCanvas.height = canvas.height
    calculateKeyPositions()
  })

  onUnmounted(() => {
    clearInterval(barInterval)
    kpsWorker.terminate()
  })
})
</script>

<!-- 内联Web Worker代码：专门算KPS，不阻塞主线程 -->
<script type="module" worker>
let presses: number[] = []
self.onmessage = () => {
  const now = Date.now()
  presses.push(now)
  // 过滤1秒内的按键，计算KPS
  presses = presses.filter(t => now - t < 1000)
  self.postMessage(presses.length)
}
</script>

## 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。
