# KeyView

<div id="kv-container">
  <div id="kv-main"></div>
  <div id="kv-keys"></div>
  <div id="kv-cfg">
    <div id="kv-cfg-top"></div>
    <div id="kv-cfg-bottom"></div>
  </div>
  <div class="kv-modal-overlay" id="kv-modal">
    <div class="kv-modal-box">
      <h3>添加按键</h3>
      <input type="text" id="kv-newKeyName" placeholder="输入按键名（如 L）" autocomplete="off">
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

<style>
/* 全部使用 VitePress 官方主题变量，自动适配亮/暗色模式 */
#kv-container {
  position: relative;
  width: 100%;
  height: 520px;
  background: var(--vp-c-bg);
  border: 1px solid var(--vp-c-divider);
  border-radius: 8px;
  overflow: hidden;
  touch-action: manipulation;
  user-select: none;
  -webkit-user-select: none;
  margin: 24px 0;
}

#kv-container #kv-main { position: absolute; inset: 0; bottom: 200px; }
#kv-container #kv-keys {
  position: absolute; left: 0; right: 0; bottom: 200px;
  display: flex; flex-direction: column; align-items: center; justify-content: flex-end;
  gap: 14px; padding: 10px 0; pointer-events: none;
}

#kv-container .kv-row { display: flex; gap: 12px; pointer-events: auto; }
#kv-container .key {
  min-width: 50px; height: 50px; border: 2px solid var(--vp-c-divider); border-radius: 8px;
  font-weight: bold; display: flex; flex-direction: column; align-items: center; justify-content: center;
  font-size: 15px; background: var(--vp-c-bg-soft); color: var(--vp-c-text-1);
  transition: background-color 0.05s, transform 0.05s, border-color 0.15s; cursor: pointer; position: relative; z-index: 1;
}
#kv-container .key.pressed {
  background: var(--vp-c-brand-dimm); border-color: var(--vp-c-brand); transform: scale(0.96); z-index: 2;
}

#kv-container .info-key {
  background: var(--vp-c-bg-mute) !important; border-color: var(--vp-c-divider-light) !important;
  color: var(--vp-c-text-2) !important; cursor: default; min-width: 110px; pointer-events: none;
}
#kv-container .info-key .kv-val { font-size: 22px; font-weight: bold; }
#kv-container .kps-key .kv-val { color: #ff9f43; }
#kv-container .total-key .kv-val { color: #00b96b; }
#kv-container .kv-lbl { font-size: 9px; color: var(--vp-c-text-3); }

#kv-container #kv-cfg {
  position: absolute; bottom: 0; left: 0; right: 0; height: 200px;
  background: var(--vp-c-bg-soft); border-top: 1px solid var(--vp-c-divider);
  display: flex; flex-direction: column;
}
#kv-container #kv-cfg-top {
  flex: 1; min-height: 0; display: flex; align-items: flex-start; padding: 10px 12px; gap: 12px;
  overflow-y: auto; overflow-x: auto;
}
#kv-container #kv-cfg-bottom {
  height: 80px; display: flex; align-items: center; padding: 0 12px; gap: 16px;
  overflow-x: auto; overflow-y: hidden; flex-shrink: 0;
}

#kv-container .cfg-item {
  display: flex; flex-direction: column; align-items: center; min-width: 72px; max-width: 90px; gap: 3px;
  position: relative; padding: 6px 4px 4px; border: 1px solid var(--vp-c-divider); border-radius: 6px;
  background: var(--vp-c-bg-mute); flex-shrink: 0;
}
#kv-container .cfg-item .del-btn {
  position: absolute; top: -5px; right: -5px; width: 15px; height: 15px; border-radius: 50%;
  background: var(--vp-c-danger); color: var(--vp-c-white); font-size: 10px;
  display: flex; align-items: center; justify-content: center; cursor: pointer; border: none;
  z-index: 10; opacity: 0.7; line-height: 1;
}
#kv-container .cfg-item .del-btn:hover { opacity: 1; }
#kv-container .cfg-item .nm { font-size: 12px; color: var(--vp-c-text-1); font-weight: bold; }
#kv-container .cfg-item .row-btns { display: flex; gap: 2px; margin-bottom: 2px; overflow-x: auto; max-width: 100%; padding-bottom: 2px; scrollbar-width: none; }
#kv-container .cfg-item .row-btns::-webkit-scrollbar { display: none; }
#kv-container .cfg-item .row-btn {
  width: 16px; height: 16px; border-radius: 2px; border: 1px solid var(--vp-c-divider); cursor: pointer;
  background: var(--vp-c-bg-soft); flex-shrink: 0; display: flex; align-items: center; justify-content: center;
  font-size: 8px; color: var(--vp-c-text-2); transition: all 0.15s;
}
#kv-container .cfg-item .row-btn:hover { background: var(--vp-c-bg-mute); }
#kv-container .cfg-item .row-btn.on { border-color: var(--vp-c-brand); background: var(--vp-c-brand-dimm); color: var(--vp-c-brand-dark); }
#kv-container .cfg-item .span-btns { display: flex; gap: 1px; margin-bottom: 2px; }
#kv-container .cfg-item .span-btn {
  width: 14px; height: 14px; border-radius: 2px; border: 1px solid var(--vp-c-divider); cursor: pointer;
  font-size: 7px; color: var(--vp-c-text-3); display: flex; align-items: center; justify-content: center;
  background: var(--vp-c-bg-soft); transition: all 0.15s;
}
#kv-container .cfg-item .span-btn.on { border-color: var(--vp-c-brand); color: var(--vp-c-brand-dark); background: var(--vp-c-brand-dimm); }
#kv-container .cfg-item .row-info { font-size: 8px; color: var(--vp-c-text-3); text-align: center; white-space: nowrap; }

#kv-container .add-btns { display: flex; flex-direction: column; gap: 6px; margin-left: 12px; padding-left: 12px; border-left: 1px solid var(--vp-c-divider); align-self: flex-start; flex-shrink: 0; }
#kv-container .add-btn { padding: 8px 14px; border-radius: 6px; font-size: 12px; font-weight: bold; cursor: pointer; border: 2px solid var(--vp-c-brand); background: var(--vp-c-bg-soft); color: var(--vp-c-brand); white-space: nowrap; transition: opacity 0.15s; }
#kv-container .add-btn:hover { opacity: 0.8; }

#kv-container .row-config { display: flex; flex-direction: column; gap: 3px; min-width: 90px; padding: 4px 6px; border: 1px solid var(--vp-c-divider); border-radius: 4px; flex-shrink: 0; }
#kv-container .row-config .rc-label { font-size: 10px; color: var(--vp-c-text-2); margin-bottom: 1px; }
#kv-container .row-config .rc-row { display: flex; align-items: center; gap: 6px; }
#kv-container .row-config .rc-label-w { font-size: 9px; color: var(--vp-c-text-3); width: 18px; }
#kv-container .row-config input[type="range"] { width: 48px; height: 12px; }
#kv-container .row-config input[type="color"] { width: 36px; height: 14px; border: none; background: none; }

#kv-container .kv-bar { position: absolute; border-radius: 2px; pointer-events: none; will-change: height, bottom, opacity; min-height: 2px; }

/* 弹窗样式 */
#kv-container .kv-modal-overlay { display: none; position: fixed; inset: 0; background: rgba(0, 0, 0, 0.6); z-index: 99999; align-items: center; justify-content: center; }
#kv-container .kv-modal-overlay.show { display: flex; }
#kv-container .kv-modal-box {
  background: var(--vp-c-bg-elv); border: 1px solid var(--vp-c-divider); border-radius: 12px;
  padding: 20px; width: 320px; max-width: 90vw; box-shadow: var(--vp-shadow-3);
}
#kv-container .kv-modal-box h3 { color: var(--vp-c-brand); margin-bottom: 14px; font-size: 16px; text-align: center; }
#kv-container .kv-modal-box input[type="text"] { width: 100%; padding: 10px; border-radius: 6px; border: 1px solid var(--vp-c-divider); background: var(--vp-c-bg-soft); color: var(--vp-c-text-1); font-size: 14px; margin-bottom: 14px; outline: none; }
#kv-container .kv-modal-box input[type="text"]:focus { border-color: var(--vp-c-brand); }
#kv-container .kv-select-group { margin-bottom: 14px; }
#kv-container .kv-select-group > label { font-size: 10px; color: var(--vp-c-text-2); display: block; margin-bottom: 6px; }
#kv-container .kv-type-btns { display: flex; gap: 6px; }
#kv-container .kv-type-btn { flex: 1; padding: 8px; border-radius: 4px; border: 2px solid var(--vp-c-divider); cursor: pointer; font-size: 11px; color: var(--vp-c-text-2); text-align: center; background: var(--vp-c-bg-soft); transition: all 0.15s; }
#kv-container .kv-type-btn.on { border-color: var(--vp-c-brand); color: var(--vp-c-brand-dark); background: var(--vp-c-brand-dimm); }
#kv-container .kv-type-btn.kps.on { border-color: #ff9f43; background: rgba(255, 159, 67, 0.1); color: #ff9f43; }
#kv-container .kv-type-btn.total.on { border-color: #00b96b; background: rgba(0, 185, 107, 0.1); color: #00b96b; }
#kv-container .kv-row-btns { display: flex; gap: 6px; flex-wrap: wrap; }
#kv-container .kv-row-btns div { padding: 6px 10px; border-radius: 4px; border: 2px solid var(--vp-c-divider); cursor: pointer; font-size: 11px; color: var(--vp-c-text-2); background: var(--vp-c-bg-soft); transition: all 0.15s; }
#kv-container .kv-row-btns div.on { border-color: var(--vp-c-brand); color: var(--vp-c-brand-dark); background: var(--vp-c-brand-dimm); }
#kv-container .kv-actions { display: flex; gap: 10px; margin-top: 4px; }
#kv-container .kv-btn { flex: 1; padding: 10px; border-radius: 6px; font-size: 13px; font-weight: bold; cursor: pointer; border: none; transition: opacity 0.15s; }
#kv-container .kv-btn.cancel { background: var(--vp-c-bg-mute); color: var(--vp-c-text-1); }
#kv-container .kv-btn.ok { background: var(--vp-c-brand); color: var(--vp-c-white); }
#kv-container .kv-btn:hover { opacity: 0.85; }
</style>

<script setup>
import { onMount } from 'vitepress'

onMount(() => {
  /* ========== 配置 ========== */
  const SAVE_KEY = 'keyview_fghj' // 固定key，不用location，避免SSR报错
  const BAR_SPEED = 300
  const BAR_FADE_DELAY = 1000
  const BAR_FADE_DUR = 2000

  /* ========== 基础数据 ========== */
  // 4行配置
  const ROWS = [
    { id: 0, width: 52, color: '#ff3366' },
    { id: 1, width: 40, color: '#ff9f43' },
    { id: 2, width: 28, color: '#ffcc00' },
    { id: 3, width: 16, color: '#00ff88' }
  ]

  // 初始布局：第一行FGHJ，第二行KPS+TOTAL
  let keys = [
    { id: 1, label: 'F', code: 'KeyF', rowId: 0, cnt: 0, type: 'normal', span: 1 },
    { id: 2, label: 'G', code: 'KeyG', rowId: 0, cnt: 0, type: 'normal', span: 1 },
    { id: 3, label: 'H', code: 'KeyH', rowId: 0, cnt: 0, type: 'normal', span: 1 },
    { id: 4, label: 'J', code: 'KeyJ', rowId: 0, cnt: 0, type: 'normal', span: 1 },
    { id: 5, label: 'KPS', code: 'KPS_INFO', rowId: 1, cnt: 0, type: 'kps', span: 2 },
    { id: 6, label: 'TOTAL', code: 'TOTAL_INFO', rowId: 1, cnt: 0, type: 'total', span: 2 }
  ]
  let nextKeyId = 7

  /* ========== 运行时状态 ========== */
  let physicalPressed = new Set()
  let active = {}
  let kpsArr = []
  let rafId = null
  let firstRowBottom = 0
  let mainRect = null
  let barId = 0
  let kpsTimer = null

  /* ========== DOM快捷查询 ========== */
  const $ = sel => document.querySelector(`#kv-container ${sel}`)
  const $$ = sel => document.querySelectorAll(`#kv-container ${sel}`)

  /* ========== 工具函数 ========== */
  // 本地存储
  const save = () => {
    try {
      localStorage.setItem(SAVE_KEY, JSON.stringify({ ROWS, keys, nextKeyId }))
    } catch {}
  }

  const load = () => {
    try {
      const d = JSON.parse(localStorage.getItem(SAVE_KEY))
      if (d) {
        ROWS.splice(0, ROWS.length, ...d.ROWS)
        keys.splice(0, keys.length, ...d.keys)
        nextKeyId = d.nextKeyId
      }
    } catch {}
  }

  // 计算第一行按键底部位置
  const calcBottom = () => {
    const main = $('#kv-main')
    if (!main) return
    mainRect = main.getBoundingClientRect()
    const firstKey = $('.key:not(.info-key)')
    if (!firstKey) {
      firstRowBottom = mainRect.height * 0.5
      return
    }
    let el = firstKey, top = 0
    while (el && el !== document.body) {
      top += el.offsetTop || 0
      el = el.offsetParent
    }
    firstRowBottom = mainRect.height - top
    if (firstRowBottom < 20) firstRowBottom = 20
    if (firstRowBottom > mainRect.height - 10) firstRowBottom = mainRect.height - 10
  }

  // 动画循环
  const tick = (now) => {
    let has = false
    for (const id in active) {
      const h = active[id]
      const dt = now - h.lastTime
      h.lastTime = now

      if (h.state === 'growing') {
        h.height += BAR_SPEED * dt / 1000
        h.bar.style.height = Math.max(h.height, 2) + 'px'
        h.bar.style.bottom = firstRowBottom + 'px'
        has = true
      } else if (h.state === 'rising') {
        h.riseT += dt
        if (h.riseT < BAR_FADE_DELAY) {
          h.bar.style.bottom = (firstRowBottom + BAR_SPEED * h.riseT / 1000) + 'px'
          has = true
        } else {
          h.state = 'fading'
          h.fadeT = 0
          h.riseBefore = BAR_SPEED * (BAR_FADE_DELAY / 1000)
        }
      } else if (h.state === 'fading') {
        h.fadeT += dt
        if (h.fadeT < BAR_FADE_DUR) {
          h.bar.style.bottom = (firstRowBottom + h.riseBefore + BAR_SPEED * h.fadeT / 1000) + 'px'
          h.bar.style.opacity = (1 - h.fadeT / BAR_FADE_DUR).toString()
          has = true
        } else {
          h.bar.remove()
          delete active[id]
        }
      }
    }
    rafId = has ? requestAnimationFrame(tick) : null
  }

  // 生成上升条
  const spawnBar = (keyId) => {
    const k = keys.find(x => x.id === keyId)
    if (!k) return
    const row = ROWS.find(r => r.id === k.rowId)
    const sample = $(`.key[data-id="${keyId}"]`)
    if (!sample || !mainRect) return
    const kr = sample.getBoundingClientRect()

    const bar = document.createElement('div')
    bar.className = 'kv-bar'
    bar.style.cssText = `
      width:${row.width}px;height:2px;background:${row.color};
      box-shadow:0 0 6px ${row.color},0 0 12px ${row.color}66;
      left:${kr.left + kr.width/2 - row.width/2 - mainRect.left}px;
      bottom:${firstRowBottom}px;opacity:1;border-radius:2px;pointer-events:none;
    `
    $('#kv-main').appendChild(bar)

    active[++barId] = {
      bar, keyId, state: 'growing', lastTime: performance.now(),
      height: 2, riseT: 0, fadeT: 0, riseBefore: 0
    }
    if (!rafId) rafId = requestAnimationFrame(tick)
  }

  // 键盘事件处理
  const triggerDown = (code) => {
    if (physicalPressed.has(code)) return
    physicalPressed.add(code)
    
    keys.filter(k => k.code === code && k.type === 'normal').forEach(k => {
      k.cnt++
      $$(`.key[data-id="${k.id}"] .kv-cnt`).forEach(el => el.innerText = k.cnt)
      spawnBar(k.id)
    })
    
    kpsArr.push(Date.now())
    save()
    syncHighlight(code)
  }

  const triggerUp = (code) => {
    if (!physicalPressed.has(code)) return
    physicalPressed.delete(code)
    
    keys.filter(k => k.code === code).forEach(k => {
      for (const id in active) {
        if (active[id].keyId === k.id && active[id].state === 'growing') {
          active[id].state = 'rising'
          active[id].riseT = 0
        }
      }
    })
    syncHighlight(code)
  }

  const syncHighlight = (code) => {
    keys.filter(k => k.code === code).forEach(k => {
      const pressed = physicalPressed.has(code)
      $$(`.key[data-id="${k.id}"]`).forEach(el => el.classList.toggle('pressed', pressed))
    })
  }

  // 渲染按键
  const renderKeys = () => {
    const c = $('#kv-keys')
    if (!c) return
    c.innerHTML = ''
    const map = {}
    keys.forEach(k => (map[k.rowId] = map[k.rowId] || []).push(k))
    
    Object.keys(map).sort().forEach(rid => {
      const rd = document.createElement('div')
      rd.className = 'kv-row'
      rd.style.zIndex = 100 - rid
      map[rid].forEach(k => {
        const d = document.createElement('div')
        d.className = 'key'
        d.dataset.code = k.code
        d.dataset.id = k.id
        d.style.width = (50 + (k.span - 1) * 56) + 'px'
        
        if (k.type === 'kps') {
          d.innerHTML = '<span class="kv-lbl">KPS</span><span class="kv-val">0</span>'
          d.classList.add('info-key', 'kps-key')
        } else if (k.type === 'total') {
          d.innerHTML = '<span class="kv-lbl">TOTAL</span><span class="kv-val">0</span>'
          d.classList.add('info-key', 'total-key')
        } else {
          d.innerHTML = `${k.label}<span class="kv-cnt">${k.cnt || 0}</span>`
          ;['mousedown', 'touchstart'].forEach(ev => 
            d.addEventListener(ev, () => triggerDown(k.code), { passive: false })
          )
          ;['mouseup', 'touchend', 'touchcancel', 'mouseleave'].forEach(ev =>
            d.addEventListener(ev, () => triggerUp(k.code), { passive: false })
          )
        }
        rd.appendChild(d)
      })
      c.appendChild(rd)
    })
    updateInfo()
    requestAnimationFrame(() => requestAnimationFrame(calcBottom))
  }

  // 更新KPS/TOTAL
  const updateInfo = () => {
    const now = Date.now()
    kpsArr = kpsArr.filter(t => now - t < 1000)
    const kps = kpsArr.length
    const tot = keys.reduce((s, k) => s + (k.cnt || 0), 0)
    $$('.kps-key .kv-val').forEach(el => el.innerText = kps)
    $$('.total-key .kv-val').forEach(el => el.innerText = tot)
  }

  // 渲染配置面板
  const renderCfg = () => {
    const top = $('#kv-cfg-top')
    if (!top) return
    top.innerHTML = ''
    
    keys.forEach(k => {
      const row = ROWS.find(r => r.id === k.rowId)
      const item = document.createElement('div')
      item.className = 'cfg-item'
      item.innerHTML = `
        <button class="del-btn" data-id="${k.id}">×</button>
        <span class="nm">${k.label}</span>
        <div class="row-btns">${ROWS.map(r => `<div class="row-btn ${k.rowId===r.id?'on':''}" data-id="${k.id}" data-rid="${r.id}">${r.id+1}</div>`).join('')}</div>
        <div class="span-btns">${[1,2,3,4].map(s => `<div class="span-btn ${k.span===s?'on':''}" data-id="${k.id}" data-span="${s}">${s}</div>`).join('')}</div>
        <div class="row-info">行${row.id+1}·${row.width}px·${k.cnt}次</div>
      `
      top.appendChild(item)
    })

    // 事件绑定
    top.querySelectorAll('.del-btn').forEach(btn => 
      btn.addEventListener('click', e => {
        e.stopPropagation()
        keys = keys.filter(k => k.id !== +btn.dataset.id)
        save()
        renderKeys()
        renderCfg()
      })
    )

    top.querySelectorAll('.row-btn').forEach(btn =>
      btn.addEventListener('click', e => {
        e.stopPropagation()
        const kid = +btn.dataset.id, rid = +btn.dataset.rid
        keys.forEach(k => k.id === kid && (k.rowId = rid))
        save()
        renderKeys()
        renderCfg()
      })
    )

    top.querySelectorAll('.span-btn').forEach(btn =>
      btn.addEventListener('click', e => {
        e.stopPropagation()
        const kid = +btn.dataset.id, s = +btn.dataset.span
        keys.forEach(k => k.id === kid && (k.span = s))
        save()
        renderKeys()
        renderCfg()
      })
    )

    // 添加按键按钮
    const addBtns = document.createElement('div')
    addBtns.className = 'add-btns'
    addBtns.innerHTML = '<button class="add-btn">+ 键</button>'
    addBtns.querySelector('.add-btn').addEventListener('click', openModal)
    top.appendChild(addBtns)

    // 底部行配置
    const bot = $('#kv-cfg-bottom')
    bot.innerHTML = ''
    ROWS.forEach(row => {
      const rc = document.createElement('div')
      rc.className = 'row-config'
      rc.innerHTML = `
        <div class="rc-label" style="color:${row.color}">第${row.id+1}行</div>
        <div class="rc-row"><span class="rc-label-w">宽</span><input type="range" min="8" max="56" value="${row.width}" data-rid="${row.id}"></div>
        <div class="rc-row"><span class="rc-label-w">色</span><input type="color" value="${row.color}" data-rid="${row.id}"></div>
      `
      bot.appendChild(rc)
    })

    bot.querySelectorAll('input[type="range"]').forEach(input =>
      input.addEventListener('input', function() {
        const r = ROWS.find(r => r.id == this.dataset.rid)
        if (r) { r.width = +this.value; save() }
      })
    )

    bot.querySelectorAll('input[type="color"]').forEach(input =>
      input.addEventListener('input', function() {
        const r = ROWS.find(r => r.id == this.dataset.rid)
        if (r) { r.color = this.value; save(); renderCfg() }
      })
    )
  }

  // 弹窗逻辑
  const openModal = () => {
    const rb = $('#kv-rowSelectBtns')
    rb.innerHTML = ROWS.map(r => `<div class="kv-row-btns" data-rid="${r.id}">第${r.id+1}行</div>`).join('')
    rb.querySelectorAll('div').forEach(btn => 
      btn.addEventListener('click', e => {
        e.preventDefault()
        rb.querySelectorAll('div').forEach(x => x.classList.remove('on'))
        btn.classList.add('on')
      })
    )
    rb.querySelector('div').classList.add('on')

    $$('.kv-type-btn').forEach(btn =>
      btn.addEventListener('click', e => {
        e.preventDefault()
        $$('.kv-type-btn').forEach(x => x.classList.remove('on'))
        btn.classList.add('on')
      })
    )
    $('.kv-type-btn.normal').classList.add('on')

    $('#kv-newKeyName').value = ''
    $('#kv-modal').classList.add('show')
    setTimeout(() => $('#kv-newKeyName').focus(), 50)
  }

  $('#kv-modalOk').addEventListener('click', e => {
    e.preventDefault()
    const label = $('#kv-newKeyName').value.trim() || 'N'
    const type = $('.kv-type-btn.on').dataset.type
    const rowId = +$('.kv-row-btns .on').dataset.rid
    let code = ''
    if (type === 'kps') code = 'KPS_' + nextKeyId
    else if (type === 'total') code = 'TOTAL_' + nextKeyId
    else code = 'Key' + label.toUpperCase()
    
    keys.push({ id: nextKeyId++, label, code, rowId, cnt: 0, type, span: 1 })
    save()
    renderKeys()
    renderCfg()
    $('#kv-modal').classList.remove('show')
  })

  $('#kv-modalCancel').addEventListener('click', e => {
    e.preventDefault()
    $('#kv-modal').classList.remove('show')
  })

  /* ========== 事件监听 ========== */
  // 全局键盘捕获（不干扰VitePress原生快捷键）
  const keydownHandler = (e) => {
    if ($('#kv-modal').classList.contains('show')) {
      if (e.key === 'Escape') {
        e.preventDefault()
        $('#kv-modal').classList.remove('show')
      }
      return
    }
    const isInput = ['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName) || document.activeElement.isContentEditable
    if (!isInput) e.preventDefault()
    triggerDown(e.code)
  }

  const keyupHandler = (e) => {
    if ($('#kv-modal').classList.contains('show')) return
    const isInput = ['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName) || document.activeElement.isContentEditable
    if (!isInput) e.preventDefault()
    triggerUp(e.code)
  }

  window.addEventListener('keydown', keydownHandler, true)
  window.addEventListener('keyup', keyupHandler, true)

  // 失焦清理
  const blurHandler = () => {
    physicalPressed.forEach(code => triggerUp(code))
    physicalPressed.clear()
    $$('.key.pressed').forEach(el => el.classList.remove('pressed'))
    for (const id in active) {
      if (active[id].state === 'growing') {
        active[id].state = 'rising'
        active[id].riseT = 0
      }
    }
  }
  window.addEventListener('blur', blurHandler)

  // KPS定时更新
  kpsTimer = setInterval(updateInfo, 200)

  // 窗口 resize
  window.addEventListener('resize', () => requestAnimationFrame(calcBottom))

  /* ========== 初始化 ========== */
  load()
  renderKeys()
  renderCfg()

  /* ========== 清理函数（页面卸载时自动执行） ========== */
  return () => {
    window.removeEventListener('keydown', keydownHandler, true)
    window.removeEventListener('keyup', keyupHandler, true)
    window.removeEventListener('blur', blurHandler)
    window.removeEventListener('resize', calcBottom)
    clearInterval(kpsTimer)
    if (rafId) cancelAnimationFrame(rafId)
  }
})
</script>

# 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。