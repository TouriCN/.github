# KeyView

<div id="kv-container">
  <div id="kv-main">
    <div v-for="bar in bars" :key="bar.id" class="kv-bar" :style="bar.style"></div>
  </div>

  <div id="kv-keys">
    <div v-for="rowKeys in groupedKeys" :key="rowKeys[0].rowId" class="kv-row">
      <div
        v-for="key in rowKeys"
        :key="key.id"
        class="key"
        :data-code="key.code"
        :class="{ 'pressed': pressedKeys.has(key.code) }"
        :style="{ width: `${50 + (key.span - 1) * 56}px` }"
        @mousedown="handleKeyDown(key.code)"
        @mouseup="handleKeyUp(key.code)"
        @touchstart.prevent="handleKeyDown(key.code)"
        @touchend.prevent="handleKeyUp(key.code)"
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
          >{{ row.id + 1 }}</div>
        </div>
        <div class="span-btns">
          <div
            v-for="span in [1,2,3,4]"
            :key="span"
            class="span-btn"
            :class="{ on: key.span === span }"
            @click="changeKeySpan(key.id, span)"
          >{{ span }}</div>
        </div>
      </div>
      <div class="add-btns">
        <button class="add-btn" @click="showModal = true">+ 键</button>
      </div>
    </div>
    <div id="kv-cfg-bottom">
      <div v-for="row in ROWS" :key="row.id" class="row-config">
        <div class="rc-label" :style="{ color: row.color }">第{{ row.id + 1 }}行</div>
        <div class="rc-row">
          <span class="rc-label-w">宽</span>
          <input type="range" min="8" max="56" v-model.number="row.width" @change="saveConfig">
        </div>
        <div class="rc-row">
          <span class="rc-label-w">色</span>
          <input type="color" v-model="row.color" @change="saveConfig">
        </div>
      </div>
    </div>
  </div>

  <div class="kv-modal-overlay" v-if="showModal">
    <div class="kv-modal-box">
      <h3>添加按键</h3>
      <input type="text" v-model="newKeyName" placeholder="输入按键名（如 L）" @keyup.enter="confirmAddKey">
      <div class="kv-select-group">
        <label>按键类型</label>
        <div class="kv-type-btns">
          <div class="kv-type-btn normal" :class="{ on: newKeyType === 'normal' }" @click="newKeyType = 'normal'">普通键</div>
          <div class="kv-type-btn kps" :class="{ on: newKeyType === 'kps' }" @click="newKeyType = 'kps'">KPS键</div>
          <div class="kv-type-btn total" :class="{ on: newKeyType === 'total' }" @click="newKeyType = 'total'">TOTAL键</div>
        </div>
      </div>
      <div class="kv-select-group">
        <label>绑定行</label>
        <div class="kv-row-btns">
          <div
            v-for="row in ROWS"
            :key="row.id"
            class="kv-row-btn"
            :class="{ on: newKeyRowId === row.id }"
            @click="newKeyRowId = row.id"
          >第{{ row.id + 1 }}行</div>
        </div>
      </div>
      <div class="kv-actions">
        <button class="kv-btn cancel" @click="showModal = false">取消</button>
        <button class="kv-btn ok" @click="confirmAddKey">确定</button>
      </div>
    </div>
  </div>
</div>

<style>
#kv-container, #kv-container * { box-sizing: border-box; }
#kv-container {
  width:100%; height:520px;
  background: var(--vp-c-bg, Canvas);
  border:1px solid var(--vp-c-divider, #e2e8f0);
  border-radius:8px; position:relative; overflow:hidden; margin:24px 0;
}
#kv-container #kv-main {
  position:absolute; inset:0; bottom:200px; z-index:2;
}
#kv-container #kv-keys {
  position:absolute; left:0; right:0; bottom:200px;
  display:flex; flex-direction:column; align-items:center; gap:14px;
  padding:10px 0; pointer-events:none; z-index:1;
}
#kv-container .kv-row {
  display:flex; gap:12px; pointer-events:auto; flex-wrap:wrap; justify-content:center;
}
#kv-container .key {
  min-width:50px; height:50px; border:2px solid var(--vp-c-divider, #e2e8f0);
  border-radius:8px; font-weight:bold;
  display:flex; flex-direction:column; align-items:center; justify-content:center;
  font-size:15px; background: var(--vp-c-bg-soft, CanvasContainer);
  color: var(--vp-c-text-1, CanvasText); cursor:pointer;
  transition: transform 0.05s ease, border-color 0.05s ease, background-color 0.05s ease;
  will-change: transform; transform: translateZ(0); pointer-events:auto;
}
#kv-container .key.pressed {
  transform: scale(0.96);
  border-color: var(--vp-c-brand, #3b82f6);
  background: var(--vp-c-brand-dimm, #dbeafe);
}
#kv-container .info-key {
  background: var(--vp-c-bg-mute, color-mix(in srgb, Canvas 95%, black)) !important;
  cursor:default; min-width:110px; pointer-events:none;
}
#kv-container .kps-key .kv-val { color:#ff9f43; font-size:22px; }
#kv-container .total-key .kv-val { color:#00b96b; font-size:22px; }
#kv-container .kv-lbl { font-size:9px; color:var(--vp-c-text-3,#64748b); }
#kv-container .kv-cnt { font-size:10px; color:var(--vp-c-text-3,#64748b); margin-top:2px; }

#kv-container .kv-bar {
  position:absolute; border-radius:2px;
  will-change: transform, opacity;
  animation: barRise 3s linear forwards;
  --bar-height: 52px;
  z-index:10;
}
@keyframes barRise {
  0% { height:2px; opacity:1; bottom:2px; }
  33% { height: var(--bar-height); opacity:1; }
  66% { height: var(--bar-height); opacity:1; transform: translateY(calc(-1 * var(--bar-height))); }
  100% { height:2px; opacity:0; transform: translateY(calc(-1 * var(--bar-height) * 2)); }
}

#kv-container #kv-cfg {
  position:absolute; bottom:0; left:0; right:0; height:200px;
  background: var(--vp-c-bg-soft, CanvasContainer);
  border-top:1px solid var(--vp-c-divider, #e2e8f0);
  display:flex; flex-direction:column;
}
#kv-container #kv-cfg-top {
  flex:1; display:flex; gap:12px; padding:10px; overflow:auto; flex-wrap:wrap; align-content:flex-start;
}
#kv-container #kv-cfg-bottom {
  height:80px; display:flex; gap:16px; padding:0 12px; align-items:center; overflow:auto; flex-wrap:wrap;
}
#kv-container .cfg-item {
  min-width:72px; max-width:90px; padding:6px; border:1px solid var(--vp-c-divider, #e2e8f0);
  border-radius:6px; background: var(--vp-c-bg-mute, color-mix(in srgb, Canvas 95%, black));
  display:flex; flex-direction:column; align-items:center; gap:4px; position:relative; flex-shrink:0;
}
#kv-container .row-btns { display:flex; flex-direction:row !important; gap:2px; width:100%; justify-content:center; }
#kv-container .span-btns { display:flex; flex-direction:row !important; gap:1px; width:100%; justify-content:center; }
#kv-container .row-btn,
#kv-container .span-btn {
  width:16px; height:16px; border-radius:2px; border:1px solid var(--vp-c-divider, #e2e8f0);
  background: var(--vp-c-bg-soft, CanvasContainer); cursor:pointer;
  display:flex; align-items:center; justify-content:center; flex-shrink:0;
}
#kv-container .row-btn.on,
#kv-container .span-btn.on {
  border-color: var(--vp-c-brand, #3b82f6);
  background: var(--vp-c-brand-dimm, #dbeafe);
}
#kv-container .del-btn {
  position:absolute; top:-5px; right:-5px; width:15px; height:15px; border-radius:50%;
  background: var(--vp-c-danger, #ef4444); color:white; border:none; cursor:pointer;
  font-size:10px; line-height:15px; text-align:center;
}
#kv-container .add-btn {
  padding:8px 14px; border:2px solid var(--vp-c-brand, #3b82f6);
  background: var(--vp-c-bg-soft, CanvasContainer); color: var(--vp-c-brand, #3b82f6);
  border-radius:6px; cursor:pointer; flex-shrink:0;
}
#kv-container .row-config {
  display:flex; flex-direction:column; gap:3px; min-width:90px; padding:4px 6px;
  border:1px solid var(--vp-c-divider, #e2e8f0); border-radius:4px; flex-shrink:0;
}
#kv-container .rc-row { display:flex; align-items:center; gap:6px; width:100%; }
#kv-container input[type="range"] { flex:1; max-width:60px; height:12px; }
#kv-container input[type="color"] { width:24px; height:24px; border:none; background:none; flex-shrink:0; }

#kv-container .kv-modal-overlay {
  position:fixed; inset:0; background:rgba(0,0,0,0.6); z-index:99999;
  display:flex; align-items:center; justify-content:center;
}
#kv-container .kv-modal-box {
  background: var(--vp-c-bg-elv, CanvasContainer);
  border:1px solid var(--vp-c-divider, #e2e8f0);
  border-radius:12px; padding:20px; width:320px; max-width:90vw;
}
#kv-container .kv-type-btn {
  flex:1; padding:8px; border:2px solid var(--vp-c-divider, #e2e8f0);
  border-radius:4px; cursor:pointer; font-size:11px; text-align:center;
}
#kv-container .kv-type-btn.on {
  border-color: var(--vp-c-brand, #3b82f6);
  background: var(--vp-c-brand-dimm, #dbeafe);
}
#kv-container .kv-row-btn {
  padding:6px 10px; border-radius:4px; border:2px solid var(--vp-c-divider, #e2e8f0);
  cursor:pointer; font-size:11px;
}
#kv-container .kv-row-btn.on {
  border-color: var(--vp-c-brand, #3b82f6);
  background: var(--vp-c-brand-dimm, #dbeafe);
}
#kv-container .kv-btn {
  flex:1; padding:10px; border:none; border-radius:6px; cursor:pointer; font-weight:bold;
}
#kv-container .kv-btn.cancel {
  background: var(--vp-c-bg-mute, color-mix(in srgb, Canvas 95%, black));
  color: var(--vp-c-text-1, CanvasText);
}
#kv-container .kv-btn.ok {
  background: var(--vp-c-brand, #3b82f6); color:white;
}

html.dark #kv-container { background: var(--vp-c-bg, #1a1a1a); border-color: #333; }
html.dark #kv-container .key { background: #242424; border-color: #333; color: #fff; }
html.dark #kv-container .key.pressed { background: #1e3a8a; border-color: #3b82f6; }
html.dark #kv-container #kv-cfg { background: #242424; border-color: #333; }
html.dark #kv-container .cfg-item { background: #1f1f1f; border-color: #333; }
html.dark #kv-container .row-btn,
html.dark #kv-container .span-btn { background: #242424; border-color: #333; color: #fff; }
html.dark #kv-container .kv-modal-box { background: #242424; border-color: #333; }
</style>

<script setup lang="ts">
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue'

const debounce = (fn: Function, delay: number) => {
  let timer: number | null = null
  return (...args: any[]) => {
    if (timer) clearTimeout(timer)
    timer = setTimeout(() => fn(...args), delay)
  }
}

const ROWS = reactive([
  { id: 0, width: 52, color: '#ff3366' },
  { id: 1, width: 40, color: '#ff9f43' },
  { id: 2, width: 28, color: '#ffcc00' },
  { id: 3, width: 16, color: '#00ff88' }
])

const keys = reactive([
  { id: 1, label: 'F', code: 'KeyF', rowId: 0, cnt: 0, type: 'normal', span: 1 },
  { id: 2, label: 'G', code: 'KeyG', rowId: 0, cnt: 0, type: 'normal', span: 1 },
  { id: 3, label: 'H', code: 'KeyH', rowId: 0, cnt: 0, type: 'normal', span: 1 },
  { id: 4, label: 'J', code: 'KeyJ', rowId: 0, cnt: 0, type: 'normal', span: 1 },
  { id: 5, label: 'KPS', code: 'KPS_INFO', rowId: 1, cnt: 0, type: 'kps', span: 2 },
  { id: 6, label: 'TOTAL', code: 'TOTAL_INFO', rowId: 1, cnt: 0, type: 'total', span: 2 }
])

const nextKeyId = ref(7)
const kps = ref(0)
const total = ref(0)
const pressedKeys = ref(new Set<string>())
const bars = ref<Array<{id: number, style: Record<string, string>}>>([])
const showModal = ref(false)
const newKeyName = ref('')
const newKeyType = ref<'normal' | 'kps' | 'total'>('normal')
const newKeyRowId = ref(0)
let barId = 0
let kpsRecords: number[] = []

const groupedKeys = computed(() => {
  const map: Record<number, typeof keys> = {}
  keys.forEach(key => {
    if (!map[key.rowId]) map[key.rowId] = []
    map[key.rowId].push(key)
  })
  return Object.values(map).sort((a, b) => a[0].rowId - b[0].rowId)
})

const saveConfig = debounce(() => {
  try {
    localStorage.setItem('keyview_config', JSON.stringify({
      ROWS,
      keys,
      nextKeyId: nextKeyId.value
    }))
  } catch {}
}, 500)

const loadConfig = () => {
  try {
    const saved = localStorage.getItem('keyview_config')
    if (saved) {
      const data = JSON.parse(saved)
      ROWS.splice(0, ROWS.length, ...data.ROWS)
      keys.splice(0, keys.length, ...data.keys)
      nextKeyId.value = data.nextKeyId
      total.value = keys.reduce((s, k) => s + k.cnt, 0)
    }
  } catch {}
}

const handleKeyDown = (code: string) => {
  try {
    if (pressedKeys.value.has(code)) return
    pressedKeys.value.add(code)

    keys.forEach(key => {
      if (key.code === code && key.type === 'normal') {
        key.cnt++
        total.value++

        // 安全获取DOM元素，避免报错打断计数
        const keyEl = document.querySelector<HTMLElement>(`#kv-container .key[data-code="${code}"]`)
        const mainEl = document.querySelector<HTMLElement>('#kv-container #kv-main')
        if (!keyEl || !mainEl) return

        const keyRect = keyEl.getBoundingClientRect()
        const mainRect = mainEl.getBoundingClientRect()
        const row = ROWS[key.rowId]

        bars.value.push({
          id: ++barId,
          style: {
            '--bar-height': `${row.width}px`,
            width: `${row.width}px`,
            background: row.color,
            left: `${keyRect.left - mainRect.left + (keyRect.width - row.width) / 2}px`,
            bottom: '2px'
          }
        })

        setTimeout(() => {
          bars.value = bars.value.filter(b => b.id !== barId)
        }, 3000)
      }
    })

    kpsRecords.push(Date.now())
    saveConfig()
  } catch (err) {
    console.error('按键按下出错:', err)
  }
}

const handleKeyUp = (code: string) => {
  pressedKeys.value.delete(code)
}

const deleteKey = (id: number) => {
  const idx = keys.findIndex(k => k.id === id)
  if (idx > -1) keys.splice(idx, 1)
  saveConfig()
}

const changeKeyRow = (id: number, rowId: number) => {
  const key = keys.find(k => k.id === id)
  if (key) key.rowId = rowId
  saveConfig()
}

const changeKeySpan = (id: number, span: number) => {
  const key = keys.find(k => k.id === id)
  if (key) key.span = span
  saveConfig()
}

const confirmAddKey = () => {
  const label = newKeyName.value.trim() || 'N'
  const code =
    newKeyType.value === 'kps' ? `KPS_${nextKeyId.value}` :
    newKeyType.value === 'total' ? `TOTAL_${nextKeyId.value}` :
    `Key${label.toUpperCase()}`

  keys.push({
    id: nextKeyId.value++,
    label,
    code,
    rowId: newKeyRowId.value,
    cnt: 0,
    type: newKeyType.value,
    span: 1
  })

  saveConfig()
  showModal.value = false
  newKeyName.value = ''
  newKeyType.value = 'normal'
  newKeyRowId.value = 0
}

const updateKPS = () => {
  const now = Date.now()
  kpsRecords = kpsRecords.filter(t => now - t < 1000)
  kps.value = kpsRecords.length
  requestAnimationFrame(updateKPS)
}

onMounted(() => {
  loadConfig()
  requestAnimationFrame(updateKPS)

  // ✅ 核心修复：加passive: false，允许preventDefault
  window.addEventListener('keydown', (e) => {
    if (showModal.value) {
      if (e.key === 'Escape') showModal.value = false
      return
    }
    const isInput = ['INPUT','TEXTAREA'].includes(document.activeElement.tagName)
    if (!isInput) {
      e.preventDefault()
      handleKeyDown(e.code)
    }
  }, { passive: false })

  window.addEventListener('keyup', (e) => {
    if (showModal.value) return
    const isInput = ['INPUT','TEXTAREA'].includes(document.activeElement.tagName)
    if (!isInput) e.preventDefault()
    handleKeyUp(e.code)
  }, { passive: false })

  window.addEventListener('blur', () => pressedKeys.value.clear())
})

onUnmounted(() => {})
</script>

## 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。
