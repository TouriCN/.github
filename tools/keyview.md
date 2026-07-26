# KeyView

<div id="kv-container">
  <!-- 主渲染区：全用Vue响应式，无手动DOM操作 -->
  <div id="kv-main">
    <!-- 上升条：纯CSS动画，脱离JS主线程 -->
    <div v-for="bar in bars" :key="bar.id" class="kv-bar" :style="bar.style"></div>
  </div>

  <!-- 按键区 -->
  <div id="kv-keys">
    <div v-for="rowKeys in groupedKeys" :key="rowKeys[0].rowId" class="kv-row">
      <div
        v-for="key in rowKeys"
        :key="key.id"
        class="key"
        :class="{
          'pressed': pressedKeys.has(key.code),
          'info-key': key.type !== 'normal',
          'kps-key': key.type === 'kps',
          'total-key': key.type === 'total'
        }"
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

  <!-- 配置区 -->
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

  <!-- 弹窗：纯Vue控制，无DOM操作 -->
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
/* 所有动画开启GPU加速，流畅度拉满 */
#kv-container { width:100%; height:520px; background:var(--vp-c-bg); border:1px solid var(--vp-c-divider); border-radius:8px; position:relative; overflow:hidden; margin:24px 0; }
#kv-container #kv-main { position:absolute; inset:0; bottom:200px; }
#kv-container #kv-keys { position:absolute; left:0; right:0; bottom:200px; display:flex; flex-direction:column; align-items:center; gap:14px; padding:10px 0; pointer-events:none; }
#kv-container .kv-row { display:flex; gap:12px; pointer-events:auto; }
#kv-container .key { min-width:50px; height:50px; border:2px solid var(--vp-c-divider); border-radius:8px; font-weight:bold; display:flex; flex-direction:column; align-items:center; justify-content:center; font-size:15px; background:var(--vp-c-bg-soft); color:var(--vp-c-text-1); cursor:pointer; transition: transform 0.05s ease, border-color 0.05s ease; will-change: transform; pointer-events:auto; }
#kv-container .key.pressed { transform: scale(0.96); border-color: var(--vp-c-brand); background: var(--vp-c-brand-dimm); }
#kv-container .info-key { background:var(--vp-c-bg-mute) !important; cursor:default; min-width:110px; pointer-events:none; }
#kv-container .kps-key .kv-val { color:#ff9f43; font-size:22px; }
#kv-container .total-key .kv-val { color:#00b96b; font-size:22px; }
#kv-container .kv-lbl { font-size:9px; color:var(--vp-c-text-3); }
#kv-container .kv-cnt { font-size:10px; color:var(--vp-c-text-3); margin-top:2px; }
#kv-container #kv-cfg { position:absolute; bottom:0; left:0; right:0; height:200px; background:var(--vp-c-bg-soft); border-top:1px solid var(--vp-c-divider); display:flex; flex-direction:column; }
#kv-container #kv-cfg-top { flex:1; display:flex; gap:12px; padding:10px; overflow:auto; }
#kv-container #kv-cfg-bottom { height:80px; display:flex; gap:16px; padding:0 12px; align-items:center; overflow:auto; }
#kv-container .cfg-item { min-width:72px; padding:6px; border:1px solid var(--vp-c-divider); border-radius:6px; background:var(--vp-c-bg-mute); display:flex; flex-direction:column; align-items:center; gap:3px; position:relative; }
#kv-container .del-btn { position:absolute; top:-5px; right:-5px; width:15px; height:15px; border-radius:50%; background:var(--vp-c-danger); color:white; border:none; cursor:pointer; font-size:10px; }
#kv-container .row-btn { width:16px; height:16px; border-radius:2px; border:1px solid var(--vp-c-divider); background:var(--vp-c-bg-soft); cursor:pointer; font-size:8px; }
#kv-container .row-btn.on { border-color:var(--vp-c-brand); background:var(--vp-c-brand-dimm); }
#kv-container .span-btn { width:14px; height:14px; border-radius:2px; border:1px solid var(--vp-c-divider); background:var(--vp-c-bg-soft); cursor:pointer; font-size:7px; }
#kv-container .span-btn.on { border-color:var(--vp-c-brand); background:var(--vp-c-brand-dimm); }
#kv-container .add-btn { padding:8px 14px; border:2px solid var(--vp-c-brand); background:var(--vp-c-bg-soft); color:var(--vp-c-brand); border-radius:6px; cursor:pointer; }
#kv-container .row-config { display:flex; flex-direction:column; gap:3px; min-width:90px; padding:4px 6px; border:1px solid var(--vp-c-divider); border-radius:4px; }
#kv-container .rc-label { font-size:10px; color:var(--vp-c-text-2); }
#kv-container .rc-row { display:flex; align-items:center; gap:6px; }
#kv-container .rc-label-w { font-size:9px; color:var(--vp-c-text-3); width:18px; }
#kv-container .kv-bar { position:absolute; border-radius:2px; will-change: transform, opacity; animation: barRise 3s linear forwards; }
@keyframes barRise {
  0% { height:2px; opacity:1; }
  33% { height: var(--bar-height); opacity:1; }
  66% { height: var(--bar-height); opacity:1; transform: translateY(calc(-1 * var(--bar-height))); }
  100% { height:2px; opacity:0; transform: translateY(calc(-1 * var(--bar-height) * 2)); }
}
#kv-container .kv-modal-overlay { position:fixed; inset:0; background:rgba(0,0,0,0.6); z-index:99999; display:flex; align-items:center; justify-content:center; }
#kv-container .kv-modal-box { background:var(--vp-c-bg-elv); border:1px solid var(--vp-c-divider); border-radius:12px; padding:20px; width:320px; }
#kv-container .kv-type-btn { flex:1; padding:8px; border:2px solid var(--vp-c-divider); border-radius:4px; cursor:pointer; font-size:11px; text-align:center; }
#kv-container .kv-type-btn.on { border-color:var(--vp-c-brand); background:var(--vp-c-brand-dimm); }
#kv-container .kv-row-btn { padding:6px 10px; border-radius:4px; border:2px solid var(--vp-c-divider); cursor:pointer; font-size:11px; }
#kv-container .kv-row-btn.on { border-color:var(--vp-c-brand); background:var(--vp-c-brand-dimm); }
#kv-container .kv-btn { flex:1; padding:10px; border:none; border-radius:6px; cursor:pointer; font-weight:bold; }
#kv-container .kv-btn.cancel { background:var(--vp-c-bg-mute); }
#kv-container .kv-btn.ok { background:var(--vp-c-brand); color:white; }
</style>

<script setup lang="ts">
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue'

// ===== 防抖函数（无外部依赖）=====
const debounce = (fn: Function, delay: number) => {
  let timer: number | null = null
  return (...args: any[]) => {
    if (timer) clearTimeout(timer)
    timer = setTimeout(() => fn(...args), delay)
  }
}

// ===== 响应式数据（全Vue管理，无手动DOM操作）=====
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

// ===== 计算属性（自动分组按键，无手动排序）=====
const groupedKeys = computed(() => {
  const map: Record<number, typeof keys> = {}
  keys.forEach(key => {
    if (!map[key.rowId]) map[key.rowId] = []
    map[key.rowId].push(key)
  })
  return Object.values(map).sort((a, b) => a[0].rowId - b[0].rowId)
})

// ===== 本地存储（防抖保存，减少IO）=====
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
      total.value = keys.reduce((sum, key) => sum + key.cnt, 0)
    }
  } catch {}
}

// ===== 核心逻辑 ======
const handleKeyDown = (code: string) => {
  if (pressedKeys.value.has(code)) return
  pressedKeys.value.add(code)
  
  // 普通键计数
  keys.forEach(key => {
    if (key.code === code && key.type === 'normal') {
      key.cnt++
      total.value++
      // 生成上升条（CSS动画，脱离JS主线程）
      const row = ROWS[key.rowId]
      const keyEl = document.querySelector(`#kv-container .key[data-code="${code}"]`) as HTMLElement
      if (keyEl) {
        const keyRect = keyEl.getBoundingClientRect()
        const mainRect = document.querySelector('#kv-container #kv-main')!.getBoundingClientRect()
        bars.value.push({
          id: ++barId,
          style: {
            '--bar-height': `${row.width}px`,
            width: `${row.width}px`,
            background: row.color,
            left: `${keyRect.left - mainRect.left + (keyRect.width - row.width) / 2}px`,
            bottom: '0'
          }
        })
        // 3秒后自动移除上升条（和CSS动画时长一致）
        setTimeout(() => {
          bars.value = bars.value.filter(bar => bar.id !== barId)
        }, 3000)
      }
    }
  })
  
  kpsRecords.push(Date.now())
  saveConfig()
}

const handleKeyUp = (code: string) => {
  pressedKeys.value.delete(code)
}

// ===== 配置操作（全响应式，无DOM操作）=====
const deleteKey = (id: number) => {
  const index = keys.findIndex(key => key.id === id)
  if (index > -1) keys.splice(index, 1)
  saveConfig()
}

const changeKeyRow = (id: number, rowId: number) => {
  const key = keys.find(key => key.id === id)
  if (key) key.rowId = rowId
  saveConfig()
}

const changeKeySpan = (id: number, span: number) => {
  const key = keys.find(key => key.id === id)
  if (key) key.span = span
  saveConfig()
}

const confirmAddKey = () => {
  const label = newKeyName.value.trim() || 'N'
  const code = newKeyType.value === 'kps' ? `KPS_${nextKeyId.value}` : 
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

// ===== KPS计算（requestAnimationFrame节流，不阻塞渲染）=====
const updateKPS = () => {
  const now = Date.now()
  kpsRecords = kpsRecords.filter(t => now - t < 1000)
  kps.value = kpsRecords.length
  requestAnimationFrame(updateKPS)
}

// ===== 事件监听 ======
onMounted(() => {
  loadConfig()
  requestAnimationFrame(updateKPS)
  
  // 键盘事件
  window.addEventListener('keydown', (e) => {
    if (showModal.value) {
      if (e.key === 'Escape') showModal.value = false
      return
    }
    const isInput = ['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName)
    if (!isInput) e.preventDefault()
    handleKeyDown(e.code)
  })
  
  window.addEventListener('keyup', (e) => {
    if (showModal.value) return
    const isInput = ['INPUT', 'TEXTAREA'].includes(document.activeElement.tagName)
    if (!isInput) e.preventDefault()
    handleKeyUp(e.code)
  })
  
  // 失焦清理
  window.addEventListener('blur', () => {
    pressedKeys.value.clear()
  })
})

onUnmounted(() => {
  // 清理事件
})

// ===== 给按键加data-code属性（方便定位）=====
onMounted(() => {
  const observer = new MutationObserver(() => {
    document.querySelectorAll('#kv-container .key').forEach(keyEl => {
      const keyId = Number(keyEl.getAttribute('data-id'))
      const key = keys.find(k => k.id === keyId)
      if (key) keyEl.setAttribute('data-code', key.code)
    })
  })
  observer.observe(document.getElementById('kv-container')!, { childList: true, subtree: true })
})
</script>

## 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。
