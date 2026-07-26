# KeyView

<ClientOnly>
<div id="kv-container">
  <!-- 主区域 -->
  <div id="kv-main"></div>

  <!-- 按键区域：Vue响应式渲染，不再用JS动态innerHTML -->
  <div id="kv-keys">
    <div v-for="rowKeys in groupedKeys" :key="rowKeys[0]?.rowId" class="kv-row">
      <div
        v-for="key in rowKeys"
        :key="key.id"
        class="key"
        :class="{
          'info-key': key.type !== 'normal',
          'kps-key': key.type === 'kps',
          'total-key': key.type === 'total',
          'pressed': pressedKeys.has(key.code)
        }"
        :data-code="key.code"
        :data-id="key.id"
        :style="{ width: `${50 + (key.span - 1) * 56}px` }"
        @mousedown="triggerDown(key.code)"
        @touchstart.prevent="triggerDown(key.code)"
        @mouseup="triggerUp(key.code)"
        @touchend="triggerUp(key.code)"
        @touchcancel="triggerUp(key.code)"
        @mouseleave="triggerUp(key.code)"
      >
        <template v-if="key.type === 'kps'">
          <span class="kv-lbl">KPS</span>
          <span class="kv-val">{{ kps }}</span>
        </template>
        <template v-else-if="key.type === 'total'">
          <span class="kv-lbl">TOTAL</span>
          <span class="kv-val">{{ total }}</span>
        </template>
        <template v-else>
          {{ key.label }}
          <span class="kv-cnt">{{ key.cnt }}</span>
        </template>
      </div>
    </div>
  </div>

  <!-- 配置区域：响应式渲染 -->
  <div id="kv-cfg">
    <div id="kv-cfg-top">
      <div v-for="key in keys" :key="key.id" class="cfg-item">
        <button class="del-btn" @click="deleteKey(key.id)">×</button>
        <span class="nm">{{ key.label }}</span>
        <div class="row-btns">
          <div
            v-for="row in ROWS"
            :key="row.id"
            class="row-btn"
            :class="{ on: key.rowId === row.id }"
            @click="changeKeyRow(key.id, row.id)"
          >
            {{ row.id + 1 }}
          </div>
        </div>
        <div class="span-btns">
          <div
            v-for="span in [1,2,3,4]"
            :key="span"
            class="span-btn"
            :class="{ on: key.span === span }"
            @click="changeKeySpan(key.id, span)"
          >
            {{ span }}
          </div>
        </div>
        <div class="row-info">行{{ key.rowId + 1 }}·{{ ROWS[key.rowId]?.width }}px·{{ key.cnt }}次</div>
      </div>
      <div class="add-btns">
        <button class="add-btn" @click="openModal">+ 键</button>
      </div>
    </div>

    <div id="kv-cfg-bottom">
      <div v-for="row in ROWS" :key="row.id" class="row-config">
        <div class="rc-label" :style="{ color: row.color }">第{{ row.id + 1 }}行</div>
        <div class="rc-row">
          <span class="rc-label-w">宽</span>
          <input
            type="range"
            min="8"
            max="56"
            v-model.number="row.width"
            @change="saveConfig"
          >
        </div>
        <div class="rc-row">
          <span class="rc-label-w">色</span>
          <input
            type="color"
            v-model="row.color"
            @change="saveConfig"
          >
        </div>
      </div>
    </div>
  </div>

  <!-- 弹窗：用v-if控制，不用class切换 -->
  <div class="kv-modal-overlay" v-if="modalVisible">
    <div class="kv-modal-box">
      <h3>添加按键</h3>
      <input
        type="text"
        id="kv-newKeyName"
        v-model="newKeyName"
        placeholder="输入按键名（如 L）"
        autocomplete="off"
        @keyup.enter="confirmAddKey"
      >
      <div class="kv-select-group">
        <label>按键类型</label>
        <div class="kv-type-btns">
          <div
            class="kv-type-btn normal"
            :class="{ on: newKeyType === 'normal' }"
            @click="newKeyType = 'normal'"
          >
            普通键
          </div>
          <div
            class="kv-type-btn kps"
            :class="{ on: newKeyType === 'kps' }"
            @click="newKeyType = 'kps'"
          >
            KPS键
          </div>
          <div
            class="kv-type-btn total"
            :class="{ on: newKeyType === 'total' }"
            @click="newKeyType = 'total'"
          >
            TOTAL键
          </div>
        </div>
      </div>
      <div class="kv-select-group">
        <label>绑定行</label>
        <div class="kv-row-btns">
          <div
            v-for="row in ROWS"
            :key="row.id"
            class="kv-row-btns"
            :class="{ on: newKeyRowId === row.id }"
            @click="newKeyRowId = row.id"
          >
            第{{ row.id + 1 }}行
          </div>
        </div>
      </div>
      <div class="kv-actions">
        <button class="kv-btn cancel" @click="modalVisible = false">取消</button>
        <button class="kv-btn ok" @click="confirmAddKey">确定</button>
      </div>
    </div>
  </div>
</div>
</ClientOnly>

<style>
/* 你之前的CSS完全不用改，直接粘贴在这里即可 */
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
#kv-container .kv-modal-overlay {
  position: fixed; inset: 0; background: rgba(0, 0, 0, 0.6); z-index: 99999;
  display: flex; align-items: center; justify-content: center;
}
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
#kv-container .kv-type-btn { flex: 1; padding: 8px; border-radius: 4px; border:2px solid var(--vp-c-divider); cursor: pointer; font-size: 11px; color: var(--vp-c-text-2); text-align: center; background: var(--vp-c-bg-soft); transition: all 0.15s; }
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
import { ref, computed, onMounted, onUnmounted } from 'vue'

// ===== 纯常量，构建安全 =====
const SAVE_KEY = 'keyview_fghj'
const BAR_SPEED = 300
const BAR_FADE_DELAY = 1000
const BAR_FADE_DUR = 2000

// ===== 响应式数据（核心：所有DOM渲染依赖这些，不用原生innerHTML）=====
const ROWS = ref([
  { id: 0, width: 52, color: '#ff3366' },
  { id: 1, width: 40, color: '#ff9f43' },
  { id: 2, width: 28, color: '#ffcc00' },
  { id: 3, width: 16, color: '#00ff88' }
])
const keys = ref([])
const nextKeyId = ref(7)
const kps = ref(0)
const total = ref(0)
const pressedKeys = ref(new Set())
const modalVisible = ref(false)
const newKeyName = ref('')
const newKeyType = ref('normal')
const newKeyRowId = ref(0)

// ===== 计算属性：按键按行分组 =====
const groupedKeys = computed(() => {
  const map = {}
  keys.value.forEach(key => {
    if (!map[key.rowId]) map[key.rowId] = []
    map[key.rowId].push(key)
  })
  return Object.values(map).sort((a, b) => a[0].rowId - b[0].rowId)
})

// ===== 运行时状态 =====
let kpsArr = []
let activeBars = {}
let rafId = null
let firstRowBottom = 0
let mainRect = null
let barId = 0
let kpsTimer = null

// ===== 工具函数 =====
const saveConfig = () => {
  try {
    localStorage.setItem(SAVE_KEY, JSON.stringify({
      ROWS: ROWS.value,
      keys: keys.value,
      nextKeyId: nextKeyId.value
    }))
  } catch {}
}

const loadConfig = () => {
  try {
    const d = JSON.parse(localStorage.getItem(SAVE_KEY))
    if (d) {
      ROWS.value = d.ROWS || ROWS.value
      keys.value = d.keys || keys.value
      nextKeyId.value = d.nextKeyId || nextKeyId.value
      // 同步total
      total.value = keys.value.reduce((sum, k) => sum + (k.cnt || 0), 0)
    }
  } catch {}
}

// ===== 动画逻辑（唯一保留的原生DOM操作，因为是动态生成的临时元素）=====
const tick = (now) => {
  let has = false
  for (const id in activeBars) {
    const bar = activeBars[id]
    const dt = now - bar.lastTime
    bar.lastTime = now

    if (bar.state === 'growing') {
      bar.height += BAR_SPEED * dt / 1000
      bar.el.style.height = Math.max(bar.height, 2) + 'px'
      bar.el.style.bottom = firstRowBottom + 'px'
      has = true
    } else if (bar.state === 'rising') {
      bar.riseT += dt
      if (bar.riseT < BAR_FADE_DELAY) {
        bar.el.style.bottom = (firstRowBottom + BAR_SPEED * bar.riseT / 1000) + 'px'
        has = true
      } else {
        bar.state = 'fading'
        bar.fadeT = 0
        bar.riseBefore = BAR_SPEED * (BAR_FADE_DELAY / 1000)
      }
    } else if (bar.state === 'fading') {
      bar.fadeT += dt
      if (bar.fadeT < BAR_FADE_DUR) {
        bar.el.style.bottom = (firstRowBottom + bar.riseBefore + BAR_SPEED * bar.fadeT / 1000) + 'px'
        bar.el.style.opacity = (1 - bar.fadeT / BAR_FADE_DUR).toString()
        has = true
      } else {
        bar.el.remove()
        delete activeBars[id]
      }
    }
  }
  rafId = has ? requestAnimationFrame(tick) : null
}

const spawnBar = (keyId) => {
  const key = keys.value.find(k => k.id === keyId)
  if (!key || !mainRect) return
  const row = ROWS.value.find(r => r.id === key.rowId)
  const keyEl = document.querySelector(`#kv-container .key[data-id="${keyId}"]`)
  if (!keyEl) return
  const kr = keyEl.getBoundingClientRect()

  const bar = document.createElement('div')
  bar.className = 'kv-bar'
  bar.style.cssText = `
    width:${row.width}px;height:2px;background:${row.color};
    box-shadow:0 0 6px ${row.color},0 0 12px ${row.color}66;
    left:${kr.left + kr.width/2 - row.width/2 - mainRect.left}px;
    bottom:${firstRowBottom}px;opacity:1;border-radius:2px;pointer-events:none;
  `
  document.querySelector('#kv-container #kv-main').appendChild(bar)

  activeBars[++barId] = {
    el: bar,
    state: 'growing',
    lastTime: performance.now(),
    height: 2,
    riseT: 0,
    fadeT: 0,
    riseBefore: 0
  }
  if (!rafId) rafId = requestAnimationFrame(tick)
}

// ===== 键盘逻辑 =====
const triggerDown = (code) => {
  if (pressedKeys.value.has(code)) return
  pressedKeys.value.add(code)

  const normalKeys = keys.value.filter(k => k.code === code && k.type === 'normal')
  normalKeys.forEach(key => {
    key.cnt++
    total.value++
    spawnBar(key.id)
  })

  kpsArr.push(Date.now())
  saveConfig()
}

const triggerUp = (code) => {
  if (!pressedKeys.value.has(code)) return
  pressedKeys.value.delete(code)

  keys.value.filter(k => k.code === code).forEach(key => {
    for (const id in activeBars) {
      if (activeBars[id].keyId === key.id && activeBars[id].state === 'growing') {
        activeBars[id].state = 'rising'
        activeBars[id].riseT = 0
      }
    }
  })
}

// ===== 配置操作 =====
const deleteKey = (id) => {
  keys.value = keys.value.filter(k => k.id !== id)
  saveConfig()
}

const changeKeyRow = (id, rowId) => {
  const key = keys.value.find(k => k.id === id)
  if (key) {
    key.rowId = rowId
    saveConfig()
  }
}

const changeKeySpan = (id, span) => {
  const key = keys.value.find(k => k.id === id)
  if (key) {
    key.span = span
    saveConfig()
  }
}

const openModal = () => {
  newKeyName.value = ''
  newKeyType.value = 'normal'
  newKeyRowId.value = 0
  modalVisible.value = true
  setTimeout(() => document.getElementById('kv-newKeyName')?.focus(), 50)
}

const confirmAddKey = () => {
  const label = newKeyName.value.trim() || 'N'
  let code = ''
  if (newKeyType.value === 'kps') {
    code = `KPS_${nextKeyId.value}`
  } else if (newKeyType.value === 'total') {
    code = `TOTAL_${nextKeyId.value}`
  } else {
    code = `Key${label.toUpperCase()}`
  }

  keys.value.push({
    id: nextKeyId.value++,
    label,
    code,
    rowId: newKeyRowId.value,
    cnt: 0,
    type: newKeyType.value,
    span: 1
  })
  saveConfig()
  modalVisible.value = false
}

// ===== 初始化 =====
onMounted(() => {
  // 加载本地配置
  loadConfig()

  // 初始化按键：如果本地没有则加载默认
  if (keys.value.length === 0) {
    keys.value = [
      { id: 1, label: 'F', code: 'KeyF', rowId: 0, cnt: 0, type: 'normal', span: 1 },
      { id: 2, label: 'G', code: 'KeyG', rowId: 0, cnt: 0, type: 'normal', span: 1 },
      { id: 3, label: 'H', code: 'KeyH', rowId: 0, cnt: 0, type: 'normal', span: 1 },
      { id: 4, label: 'J', code: 'KeyJ', rowId: 0, cnt: 0, type: 'normal', span: 1 },
      { id: 5, label: 'KPS', code: 'KPS_INFO', rowId: 1, cnt: 0, type: 'kps', span: 2 },
      { id: 6, label: 'TOTAL', code: 'TOTAL_INFO', rowId: 1, cnt: 0, type: 'total', span: 2 }
    ]
    nextKeyId.value = 7
    saveConfig()
  }

  // 计算按键位置
  const calcBottom = () => {
    const main = document.querySelector('#kv-container #kv-main')
    if (!main) return
    mainRect = main.getBoundingClientRect()
    const firstKey = document.querySelector('#kv-container .key:not(.info-key)')
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
  requestAnimationFrame(() => requestAnimationFrame(calcBottom))
  window.addEventListener('resize', () => requestAnimationFrame(calcBottom))

  // KPS更新定时器
  kpsTimer = setInterval(() => {
    const now = Date.now()
    kpsArr = kpsArr.filter(t => now - t < 1000)
    kps.value = kpsArr.length
  }, 200)

  // 全局键盘监听
  const keydownHandler = (e) => {
    if (modalVisible.value) {
      if (e.key === 'Escape') modalVisible.value = false
      return
    }
    const isInput = ['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName) || document.activeElement.isContentEditable
    if (!isInput) e.preventDefault()
    triggerDown(e.code)
  }

  const keyupHandler = (e) => {
    if (modalVisible.value) return
    const isInput = ['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName) || document.activeElement.isContentEditable
    if (!isInput) e.preventDefault()
    triggerUp(e.code)
  }

  window.addEventListener('keydown', keydownHandler, true)
  window.addEventListener('keyup', keyupHandler, true)

  // 失焦清理
  const blurHandler = () => {
    pressedKeys.value.clear()
    for (const id in activeBars) {
      if (activeBars[id].state === 'growing') {
        activeBars[id].state = 'rising'
        activeBars[id].riseT = 0
      }
    }
  }
  window.addEventListener('blur', blurHandler)

  // 清理函数
  onUnmounted(() => {
    window.removeEventListener('keydown', keydownHandler, true)
    window.removeEventListener('keyup', keyupHandler, true)
    window.removeEventListener('blur', blurHandler)
    window.removeEventListener('resize', calcBottom)
    clearInterval(kpsTimer)
    if (rafId) cancelAnimationFrame(rafId)
  })
})
</script>

# 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。