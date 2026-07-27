# KeyView

<script setup lang="ts">
import { ref, reactive, onMounted, defineComponent, h } from 'vue'

// ===== 数据 & 逻辑（纯 JS，不依赖模板）=====
const keys = reactive([
  { id: 1, label: 'F', code: 'KeyF', rowId: 0, cnt: 0, type: 'normal', span: 1 },
  { id: 2, label: 'G', code: 'KeyG', rowId: 0, cnt: 0, type: 'normal', span: 1 },
  { id: 3, label: 'H', code: 'KeyH', rowId: 0, cnt: 0, type: 'normal', span: 1 },
  { id: 4, label: 'J', code: 'KeyJ', rowId: 0, cnt: 0, type: 'normal', span: 1 },
  { id: 5, label: 'KPS', code: 'KPS', type: 'kps', span: 2 },
  { id: 6, label: 'TOTAL', code: 'TOTAL', type: 'total', span: 2 }
])
const rows = reactive([
  { id: 0, width: 52, color: '#ff3366' },
  { id: 1, width: 40, color: '#ff9f43' },
  { id: 2, width: 28, color: '#ffcc00' },
  { id: 3, width: 16, color: '#00ff88' }
])
const nextId = ref(7)
const kps = ref(0)
const total = ref(0)
const pressed = ref(new Set<string>())
const bars = ref<any[]>([])

const loadConfig = () => {
  try {
    const data = JSON.parse(localStorage.getItem('keyview') || '{}')
    if (data.keys) keys.splice(0, keys.length, ...data.keys)
    if (data.rows) rows.splice(0, rows.length, ...data.rows)
    if (data.nextId) nextId.value = data.nextId
    total.value = keys.reduce((s, k) => s + (k.cnt || 0), 0)
  } catch {}
}

const saveConfig = () => {
  localStorage.setItem('keyview', JSON.stringify({
    keys: keys.map(k => ({ ...k })),
    rows: rows.map(r => ({ ...r })),
    nextId: nextId.value
  }))
}

const handleKeyDown = (code: string) => {
  if (pressed.value.has(code)) return
  pressed.value.add(code)
  const key = keys.find(k => k.code === code)
  if (key && key.type === 'normal') {
    key.cnt++
    total.value++
    const row = rows[key.rowId]
    bars.value.push({
      id: Date.now(),
      x: 0,
      y: 0,
      width: row.width,
      height: 2,
      color: row.color,
      alpha: 1
    })
    saveConfig()
  }
  kps.value = [...pressed.value].length
}

const handleKeyUp = (code: string) => {
  pressed.value.delete(code)
}

// 更新上升条
const updateBars = () => {
  bars.value = bars.value.filter(bar => {
    bar.y -= 3
    bar.height = Math.min(bar.height + 0.1, bar.width)
    bar.alpha -= 0.003
    return bar.y > -bar.height && bar.alpha > 0
  })
}

onMounted(() => {
  loadConfig()
  window.addEventListener('keydown', (e) => {
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName || '')) return
    e.preventDefault()
    handleKeyDown(e.code)
  })
  window.addEventListener('keyup', (e) => {
    if (['INPUT', 'TEXTAREA'].includes(document.activeElement?.tagName || '')) return
    e.preventDefault()
    handleKeyUp(e.code)
  })
  setInterval(() => { kps.value = [...pressed.value].length }, 200)
  setInterval(updateBars, 16)
})

// ===== 激进核心：纯 Render 函数组件（无模板，无 SSR 坑）=====
const KeyViewRenderer = defineComponent({
  name: 'KeyViewRenderer',
  setup() {
    // 返回 render 函数，Vue 直接调用，不经过模板编译
    return () => h('div', {
      style: {
        width: '100%',
        height: '520px',
        border: '1px solid #e2e8f0',
        borderRadius: '8px',
        margin: '24px 0',
        position: 'relative',
        overflow: 'hidden',
        background: '#ffffff'
      }
    }, [
      // 上升条层
      h('div', { style: { position: 'absolute', inset: '0 0 200px 0', zIndex: 2 } },
        bars.value.map(bar => h('div', {
          style: {
            position: 'absolute',
            left: `${bar.x}px`,
            bottom: `${bar.y}px`,
            width: `${bar.width}px`,
            height: `${bar.height}px`,
            background: bar.color,
            borderRadius: '2px',
            opacity: bar.alpha
          }
        }))
      ),
      // 按键层
      h('div', { style: { position: 'absolute', left: 0, right: 0, bottom: '200px', display: 'flex', flexDirection: 'column', alignItems: 'center', gap: '14px', padding: '10px 0', zIndex: 1 } },
        [...new Set(keys.map(k => k.rowId))].sort().map(rowId =>
          h('div', { style: { display: 'flex', gap: '12px' } },
            keys.filter(k => k.rowId === rowId).map(key =>
              h('div', {
                onMousedown: () => handleKeyDown(key.code),
                onMouseup: () => handleKeyUp(key.code),
                onTouchstart: (e: TouchEvent) => { e.preventDefault(); handleKeyDown(key.code) },
                onTouchend: (e: TouchEvent) => { e.preventDefault(); handleKeyUp(key.code) },
                style: {
                  minWidth: key.type !== 'normal' ? '110px' : '50px',
                  width: `${50 + (key.span - 1) * 56}px`,
                  height: '50px',
                  border: '2px solid #e2e8f0',
                  borderRadius: '8px',
                  display: 'flex',
                  flexDirection: 'column',
                  alignItems: 'center',
                  justifyContent: 'center',
                  fontWeight: 'bold',
                  cursor: 'pointer',
                  transform: pressed.value.has(key.code) ? 'scale(0.96)' : 'scale(1)',
                  borderColor: pressed.value.has(key.code) ? '#3b82f6' : '#e2e8f0',
                  background: pressed.value.has(key.code) ? '#dbeafe' : (key.type !== 'normal' ? '#edf2f7' : '#f6f6f7'),
                  color: '#1a202c',
                  transition: 'transform 0.05s ease'
                }
              }, key.type === 'kps' ? [
                h('span', { style: { fontSize: '9px', color: '#64748b' } }, 'KPS'),
                h('span', { style: { fontSize: '22px', color: '#ff9f43' } }, kps.value.toString())
              ] : key.type === 'total' ? [
                h('span', { style: { fontSize: '9px', color: '#64748b' } }, 'TOTAL'),
                h('span', { style: { fontSize: '22px', color: '#00b96b' } }, total.value.toString())
              ] : [
                key.label,
                h('span', { style: { fontSize: '10px', color: '#64748b', marginTop: '2px' } }, key.cnt.toString())
              ])
            )
          )
        )
      ),
      // 配置层
      h('div', { style: { position: 'absolute', bottom: 0, left: 0, right: 0, height: '200px', background: '#f6f6f7', borderTop: '1px solid #e2e8f0', display: 'flex', flexDirection: 'column' } }, [
        h('div', { style: { flex: 1, display: 'flex', gap: '12px', padding: '10px', overflow: 'auto' } }, [
          ...keys.map(key => h('div', { style: { minWidth: '72px', padding: '6px', border: '1px solid #e2e8f0', borderRadius: '6px', background: '#edf2f7', display: 'flex', flexDirection: 'column', alignItems: 'center', gap: '3px', position: 'relative' } }, [
            h('button', { onClick: () => { keys.splice(keys.findIndex(k => k.id === key.id), 1); saveConfig() }, style: { position: 'absolute', top: '-5px', right: '-5px', width: '15px', height: '15px', borderRadius: '50%', background: '#ef4444', color: 'white', border: 'none', cursor: 'pointer', fontSize: '10px' } }, '×'),
            h('span', {}, key.label),
            h('div', { style: { display: 'flex', gap: '2px' } }, rows.map(row => h('div', { onClick: () => { const k = keys.find(k => k.id === key.id); if(k) k.rowId = row.id; saveConfig() }, style: { width: '16px', height: '16px', borderRadius: '2px', border: '1px solid #e2e8f0', background: key.rowId === row.id ? '#dbeafe' : '#f6f6f7', cursor: 'pointer', fontSize: '8px', display: 'flex', alignItems: 'center', justifyContent: 'center' } }, row.id + 1))),
            h('div', { style: { display: 'flex', gap: '1px' } }, [1,2,3,4].map(span => h('div', { onClick: () => { const k = keys.find(k => k.id === key.id); if(k) k.span = span; saveConfig() }, style: { width: '14px', height: '14px', borderRadius: '2px', border: '1px solid #e2e8f0', background: key.span === span ? '#dbeafe' : '#f6f6f7', cursor: 'pointer', fontSize: '7px', display: 'flex', alignItems: 'center', justifyContent: 'center' } }, span.toString())))
          ])),
          h('button', { onClick: () => { const label = prompt('输入按键名（如L）') || 'N'; keys.push({ id: nextId.value++, label, code: `Key${label.toUpperCase()}`, rowId: 0, cnt: 0, type: 'normal', span: 1 }); saveConfig() }, style: { padding: '8px 14px', border: '2px solid #3b82f6', background: '#f6f6f7', color: '#3b82f6', borderRadius: '6px', cursor: 'pointer', flexShrink: 0 } }, '+ 键')
        ]),
        h('div', { style: { height: '80px', display: 'flex', gap: '16px', padding: '0 12px', alignItems: 'center', overflow: 'auto' } },
          rows.map(row => h('div', { style: { display: 'flex', flexDirection: 'column', gap: '3px', minWidth: '90px', padding: '4px 6px', border: '1px solid #e2e8f0', borderRadius: '4px' } }, [
            h('div', { style: { fontSize: '10px', color: row.color } }, `第${row.id + 1}行`),
            h('div', { style: { display: 'flex', alignItems: 'center', gap: '6px' } }, [h('span', { style: { fontSize: '9px', color: '#64748b', width: '18px' } }, '宽'), h('input', { type: 'range', min: 8, max: 56, value: row.width, onInput: (e: Event) => { row.width = +(e.target as HTMLInputElement).value; saveConfig() }, style: { flex: 1, maxWidth: '60px', height: '12px' } })]),
            h('div', { style: { display: 'flex', alignItems: 'center', gap: '6px' } }, [h('span', { style: { fontSize: '9px', color: '#64748b', width: '18px' } }, '色'), h('input', { type: 'color', value: row.color, onInput: (e: Event) => { row.color = (e.target as HTMLInputElement).value; saveConfig() }, style: { width: '24px', height: '24px', border: 'none', background: 'none' } })])
          ]))
        )
      ])
    ])
  }
})
</script>

<!-- 单组件出口，无第二个 script -->
<KeyViewRenderer />

## 该工具有什么用处？
可用于显示游戏中输入的按键，如果您发布视频的话，这实际上可以很方便地给观众查看手法。
