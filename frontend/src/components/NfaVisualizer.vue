<template>
  <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
    <div class="flex items-center justify-between mb-3">
      <h3 class="text-sm font-bold text-slate-400">NFA 状态机可视化</h3>
      <span v-if="store.nfa" class="text-xs text-slate-500">{{ store.nfa.states.length }} 状态 · {{ store.nfa.transitions.length }} 转移</span>
    </div>

    <!-- 视图工具栏：局部缩放 / 视野重置 / 聚焦路径 -->
    <div class="flex flex-wrap items-center gap-2 mb-2 text-xs">
      <button @click="zoomBy(1.25)" class="px-2 py-1 bg-slate-700 hover:bg-slate-600 rounded">＋ 放大</button>
      <button @click="zoomBy(0.8)" class="px-2 py-1 bg-slate-700 hover:bg-slate-600 rounded">－ 缩小</button>
      <button @click="resetView" class="px-2 py-1 bg-slate-700 hover:bg-slate-600 rounded">⟲ 重置视野</button>
      <button @click="fitToPath" :disabled="!focusInfo"
        class="px-2 py-1 rounded bg-cyan-700 hover:bg-cyan-600 disabled:opacity-40 disabled:hover:bg-cyan-700">◎ 聚焦路径</button>
      <button @click="focusEnabled = !focusEnabled"
        :class="['px-2 py-1 rounded', focusEnabled ? 'bg-cyan-900 text-cyan-300 border border-cyan-600' : 'bg-slate-700 hover:bg-slate-600']">
        路径高亮: {{ focusEnabled ? '开' : '关' }}
      </button>
      <span class="ml-auto text-slate-500 font-mono">{{ Math.round(view.scale * 100) }}%</span>
    </div>

    <!-- 可读提示：结构过大 / 节点重叠 / 无激活路径 -->
    <div v-if="hints.length" class="mb-2 space-y-1">
      <div v-for="(h, i) in hints" :key="i"
        :class="['text-xs px-2 py-1 rounded border',
          h.type === 'warn' ? 'bg-amber-900/40 text-amber-300 border-amber-700/50' : 'bg-slate-900 text-slate-400 border-slate-700']">
        {{ h.type === 'warn' ? '⚠' : 'ℹ' }} {{ h.text }}
      </div>
    </div>

    <!-- 主画布（拖拽平移 / 滚轮缩放） + 概览小地图 -->
    <div class="relative">
      <canvas ref="canvasRef"
        :class="['w-full bg-slate-900 rounded-lg border border-slate-700 select-none', panning ? 'cursor-grabbing' : 'cursor-grab']"
        @mousedown="onPanStart" @mousemove="onPanMove" @mouseup="onPanEnd" @mouseleave="onPanEnd"
        @wheel.prevent="onWheel"></canvas>
      <div v-if="store.nfa" class="absolute right-2 bottom-2 bg-slate-800/85 rounded border border-slate-600 p-1">
        <div class="text-[10px] text-slate-500 px-0.5 pb-0.5">概览</div>
        <canvas ref="miniRef" width="160" height="100" class="rounded cursor-pointer" title="点击定位视野"
          @mousedown="onMiniJump"></canvas>
      </div>
    </div>

    <div class="mt-2 flex flex-wrap gap-x-4 gap-y-1 text-xs text-slate-500">
      <span><span class="inline-block w-3 h-3 rounded-full bg-cyan-500 mr-1"></span>起始状态</span>
      <span><span class="inline-block w-3 h-3 rounded-full bg-green-500 mr-1"></span>接受状态</span>
      <span><span class="inline-block w-3 h-3 rounded-full bg-orange-500 mr-1"></span>当前激活</span>
      <span><span class="inline-block w-3 h-3 rounded-full border-2 border-cyan-400 bg-slate-700 mr-1"></span>路径状态</span>
      <span><span class="inline-block w-6 h-0.5 bg-cyan-400 mr-1 align-middle"></span>路径转移</span>
      <span><span class="inline-block w-3 h-3 rounded-full bg-slate-600 mr-1"></span>普通状态</span>
    </div>

    <!-- 路径看板：路径统计 + 状态链，与当前步骤联动 -->
    <div class="mt-3 bg-slate-900 rounded-lg border border-slate-700 p-3">
      <div class="flex flex-wrap items-center justify-between gap-2 mb-2">
        <h4 class="text-xs font-bold text-slate-400">路径看板</h4>
        <div v-if="store.matchResult" class="flex gap-3 text-[11px] text-slate-500">
          <span>路径节点 <b class="text-cyan-400">{{ pathStats.states }}</b></span>
          <span>路径转移 <b class="text-cyan-400">{{ pathStats.edges }}</b></span>
          <span>当前步骤 <b class="text-orange-400">{{ store.currentStep }}</b> / {{ store.matchResult.steps.length }}</span>
        </div>
      </div>
      <div v-if="!store.matchResult" class="text-xs text-slate-500">尚未执行匹配，无路径数据</div>
      <div v-else-if="!pathChain.length" class="text-xs text-slate-500">没有激活路径：当前输入未产生有效的状态转移</div>
      <div v-else class="flex items-center gap-1 overflow-x-auto pb-1">
        <span v-if="chainView.before > 0" class="text-[10px] text-slate-600 shrink-0">…+{{ chainView.before }}</span>
        <template v-for="(node, i) in chainView.items" :key="i">
          <button @click="jumpToStep(node.step)"
            :title="`步骤 ${node.step}` + (node.char ? ` · 字符 '${node.char}'` : '')"
            :class="['px-1.5 py-0.5 rounded font-mono text-[11px] shrink-0 border',
              node.step === store.currentStep ? 'bg-orange-600 text-white border-orange-400'
              : node.step < store.currentStep ? 'bg-cyan-900/60 text-cyan-300 border-cyan-800'
              : 'bg-slate-800 text-slate-500 border-slate-700 hover:border-slate-500']">
            {{ node.state }}
          </button>
          <span v-if="i < chainView.items.length - 1 || chainView.after > 0" class="text-slate-600 text-[10px] shrink-0">→</span>
        </template>
        <span v-if="chainView.after > 0" class="text-[10px] text-slate-600 shrink-0">…+{{ chainView.after }}</span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted, onBeforeUnmount, watch } from 'vue'
import { useRegexStore } from '../store/regex'

const store = useRegexStore()
const canvasRef = ref<HTMLCanvasElement | null>(null)
const miniRef = ref<HTMLCanvasElement | null>(null)

// ---- 视口状态：平移/缩放后切换步骤时保持不变，仅在重新执行匹配时重置 ----
const view = reactive({ x: 0, y: 0, scale: 1 })
const canvasSize = reactive({ w: 800, h: 500 })
const panning = ref(false)
const focusEnabled = ref(true)

// ---- 聚焦路径：当前步骤之前所有步骤经过的状态与转移 ----
interface FocusInfo { states: Set<number>; edges: Set<string> }
const focusInfo = computed<FocusInfo | null>(() => {
  const mr = store.matchResult
  if (!mr || !mr.steps.length) return null
  const upto = Math.min(store.currentStep, mr.steps.length - 1)
  const states = new Set<number>()
  const edges = new Set<string>()
  for (let i = 0; i <= upto; i++) {
    const st = mr.steps[i]
    if (st.currentState >= 0) states.add(st.currentState)
    if (st.nextState >= 0) states.add(st.nextState)
    if (st.currentState >= 0 && st.nextState >= 0) edges.add(st.currentState + '->' + st.nextState)
  }
  return states.size ? { states, edges } : null
})

const activeStep = computed(() => {
  const mr = store.matchResult
  if (!mr || store.currentStep >= mr.steps.length) return null
  return mr.steps[store.currentStep]
})

const pathStats = computed(() => ({
  states: focusInfo.value?.states.size ?? 0,
  edges: focusInfo.value?.edges.size ?? 0
}))

// ---- 路径看板：完整状态链（含未走过的步骤），当前位置高亮 ----
interface ChainNode { state: number; step: number; char: string }
const pathChain = computed<ChainNode[]>(() => {
  const mr = store.matchResult
  if (!mr) return []
  const chain: ChainNode[] = []
  mr.steps.forEach((st, i) => {
    if (st.currentState >= 0 && (!chain.length || chain[chain.length - 1].state !== st.currentState)) {
      chain.push({ state: st.currentState, step: i, char: '' })
    }
    if (st.nextState >= 0) chain.push({ state: st.nextState, step: i, char: st.char })
  })
  return chain
})

const chainPos = computed(() => {
  const chain = pathChain.value
  let pos = 0
  for (let i = 0; i < chain.length; i++) {
    if (chain[i].step <= store.currentStep) pos = i
    else break
  }
  return pos
})

const CHAIN_WINDOW = 24
const chainView = computed(() => {
  const chain = pathChain.value
  const max = CHAIN_WINDOW * 2 + 1
  if (chain.length <= max) return { items: chain, before: 0, after: 0 }
  const start = Math.max(0, Math.min(chainPos.value - CHAIN_WINDOW, chain.length - max))
  return { items: chain.slice(start, start + max), before: start, after: chain.length - start - max }
})

function jumpToStep(step: number) {
  store.currentStep = step
}

// ---- 可读提示 ----
const overlapCount = computed(() => {
  const nfa = store.nfa
  if (!nfa || nfa.states.length > 400) return 0
  let count = 0
  const s = nfa.states
  for (let i = 0; i < s.length; i++) {
    for (let j = i + 1; j < s.length; j++) {
      const dx = s[i].x - s[j].x, dy = s[i].y - s[j].y
      if (dx * dx + dy * dy < 42 * 42) count++
    }
  }
  return count
})

const hints = computed(() => {
  const list: { type: 'warn' | 'info'; text: string }[] = []
  const nfa = store.nfa
  if (!nfa) {
    list.push({ type: 'info', text: store.error ? '正则解析失败，修正后重新执行即可恢复画布' : '暂无可展示的状态机，请先执行匹配' })
    return list
  }
  if (nfa.states.length > 60 || nfa.transitions.length > 160) {
    list.push({ type: 'warn', text: `结构较大（${nfa.states.length} 状态 / ${nfa.transitions.length} 转移），已自动适配视野，可拖拽平移或滚轮缩放浏览` })
  }
  if (overlapCount.value > 0) {
    list.push({ type: 'warn', text: `检测到 ${overlapCount.value} 处节点重叠，可缩放画布或使用「聚焦路径」查看局部细节` })
  }
  if (!store.matchResult || !store.matchResult.steps.length) {
    list.push({ type: 'info', text: '当前没有激活路径：执行匹配后，画布将只突出相关节点与连线' })
  }
  return list
})

// ---- 视口变换 ----
function worldBBox() {
  const states = store.nfa!.states
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity
  states.forEach(n => {
    minX = Math.min(minX, n.x); minY = Math.min(minY, n.y)
    maxX = Math.max(maxX, n.x); maxY = Math.max(maxY, n.y)
  })
  return { minX, minY, maxX, maxY }
}

function fitBBox(bbox: { minX: number; minY: number; maxX: number; maxY: number }, pad = 60) {
  const w = Math.max(1, bbox.maxX - bbox.minX) + pad * 2
  const h = Math.max(1, bbox.maxY - bbox.minY) + pad * 2
  view.scale = Math.min(2, Math.max(0.1, Math.min(canvasSize.w / w, canvasSize.h / h)))
  view.x = canvasSize.w / 2 - ((bbox.minX + bbox.maxX) / 2) * view.scale
  view.y = canvasSize.h / 2 - ((bbox.minY + bbox.maxY) / 2) * view.scale
}

function resetView() {
  if (store.nfa && store.nfa.states.length) fitBBox(worldBBox())
  draw()
}

// 局部缩放：把视野适配到当前激活路径的包围盒
function fitToPath() {
  const focus = focusInfo.value
  const nfa = store.nfa
  if (!focus || !nfa) return
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity
  nfa.states.forEach(n => {
    if (!focus.states.has(n.id)) return
    minX = Math.min(minX, n.x); minY = Math.min(minY, n.y)
    maxX = Math.max(maxX, n.x); maxY = Math.max(maxY, n.y)
  })
  if (minX === Infinity) return
  fitBBox({ minX, minY, maxX, maxY }, 90)
  draw()
}

function zoomAt(cx: number, cy: number, factor: number) {
  const ns = Math.min(4, Math.max(0.1, view.scale * factor))
  const k = ns / view.scale
  view.x = cx - (cx - view.x) * k
  view.y = cy - (cy - view.y) * k
  view.scale = ns
  draw()
}

function zoomBy(factor: number) {
  zoomAt(canvasSize.w / 2, canvasSize.h / 2, factor)
}

// ---- 交互：拖拽平移 / 滚轮缩放 / 概览定位 ----
let panStart: { x: number; y: number; vx: number; vy: number } | null = null
function onPanStart(e: MouseEvent) {
  panStart = { x: e.offsetX, y: e.offsetY, vx: view.x, vy: view.y }
  panning.value = true
}
function onPanMove(e: MouseEvent) {
  if (!panStart) return
  view.x = panStart.vx + (e.offsetX - panStart.x)
  view.y = panStart.vy + (e.offsetY - panStart.y)
  draw()
}
function onPanEnd() {
  panStart = null
  panning.value = false
}
function onWheel(e: WheelEvent) {
  zoomAt(e.offsetX, e.offsetY, e.deltaY < 0 ? 1.15 : 1 / 1.15)
}

function miniTransform() {
  const nfa = store.nfa
  if (!nfa || !nfa.states.length) return null
  const bbox = worldBBox()
  const pad = 30
  const w = bbox.maxX - bbox.minX + pad * 2
  const h = bbox.maxY - bbox.minY + pad * 2
  const s = Math.min(160 / w, 100 / h)
  return { s, ox: (160 - w * s) / 2 - (bbox.minX - pad) * s, oy: (100 - h * s) / 2 - (bbox.minY - pad) * s }
}

function onMiniJump(e: MouseEvent) {
  const t = miniTransform()
  const mini = miniRef.value
  if (!t || !mini) return
  const rect = mini.getBoundingClientRect()
  const wx = (e.clientX - rect.left - t.ox) / t.s
  const wy = (e.clientY - rect.top - t.oy) / t.s
  view.x = canvasSize.w / 2 - wx * view.scale
  view.y = canvasSize.h / 2 - wy * view.scale
  draw()
}

// ---- 渲染 ----
function draw() {
  const canvas = canvasRef.value
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  if (!ctx) return
  const dpr = window.devicePixelRatio || 1
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0)
  ctx.clearRect(0, 0, canvasSize.w, canvasSize.h)

  const nfa = store.nfa
  if (!nfa) {
    ctx.fillStyle = '#64748b'
    ctx.font = '14px sans-serif'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(store.error ? '正则解析失败，请修正后重新执行' : '暂无状态机数据，点击「执行匹配」生成', canvasSize.w / 2, canvasSize.h / 2)
    drawMini()
    return
  }

  const focus = focusEnabled.value ? focusInfo.value : null
  const active = activeStep.value
  const activeStates = new Set<number>()
  if (active) {
    if (active.currentState >= 0) activeStates.add(active.currentState)
    if (active.nextState >= 0) activeStates.add(active.nextState)
  }

  ctx.save()
  ctx.translate(view.x, view.y)
  ctx.scale(view.scale, view.scale)

  // Draw transitions
  nfa.transitions.forEach(t => {
    const from = nfa.states.find(s => s.id === t.from)
    const to = nfa.states.find(s => s.id === t.to)
    if (!from || !to) return

    const isActive = activeStates.has(t.from) && activeStates.has(t.to)
    const onPath = !isActive && !!focus && focus.edges.has(t.from + '->' + t.to)
    const dimmed = !!focus && !isActive && !onPath

    ctx.globalAlpha = dimmed ? 0.12 : 1
    ctx.strokeStyle = isActive ? '#f97316' : onPath ? '#22d3ee' : '#475569'
    ctx.lineWidth = isActive ? 3 : onPath ? 2 : 1
    ctx.beginPath()
    ctx.moveTo(from.x, from.y)

    if (t.from === t.to) {
      // Self loop
      const mx = from.x, my = from.y - 35
      ctx.quadraticCurveTo(mx + 25, my, from.x + 10, from.y - 15)
    } else {
      const mx = (from.x + to.x) / 2, my = (from.y + to.y) / 2
      const dx = to.x - from.x, dy = to.y - from.y
      const len = Math.sqrt(dx * dx + dy * dy) || 1
      const offX = -dy / len * 20, offY = dx / len * 20
      ctx.quadraticCurveTo(mx + offX, my + offY, to.x, to.y)
    }
    ctx.stroke()

    // Arrowhead
    const angle = Math.atan2(to.y - from.y, to.x - from.x)
    const ex = to.x - Math.cos(angle) * 22, ey = to.y - Math.sin(angle) * 22
    ctx.beginPath()
    ctx.moveTo(ex, ey)
    ctx.lineTo(ex - Math.cos(angle - 0.4) * 8, ey - Math.sin(angle - 0.4) * 8)
    ctx.lineTo(ex - Math.cos(angle + 0.4) * 8, ey - Math.sin(angle + 0.4) * 8)
    ctx.closePath()
    ctx.fillStyle = isActive ? '#f97316' : onPath ? '#22d3ee' : '#475569'
    ctx.fill()

    // Label
    const mx = (from.x + to.x) / 2, my = (from.y + to.y) / 2
    const dx2 = to.x - from.x, dy2 = to.y - from.y
    const len2 = Math.sqrt(dx2 * dx2 + dy2 * dy2) || 1
    const lx = mx - dy2 / len2 * 15, ly = my + dx2 / len2 * 15
    ctx.fillStyle = isActive ? '#fbbf24' : onPath ? '#a5f3fc' : '#94a3b8'
    ctx.font = '11px monospace'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'alphabetic'
    ctx.fillText(t.label, lx, ly)
  })

  // Draw states
  nfa.states.forEach(s => {
    const isActive = activeStates.has(s.id)
    const onPath = !isActive && !!focus && focus.states.has(s.id)
    const dimmed = !!focus && !isActive && !onPath

    ctx.globalAlpha = dimmed ? 0.2 : 1
    const color = s.isStart ? '#06b6d4' : s.isAccept ? '#22c55e' : isActive ? '#f97316' : '#475569'

    ctx.beginPath()
    ctx.arc(s.x, s.y, 20, 0, Math.PI * 2)
    ctx.fillStyle = color
    ctx.fill()
    ctx.strokeStyle = isActive ? '#fbbf24' : onPath ? '#22d3ee' : '#1e293b'
    ctx.lineWidth = isActive || onPath ? 2.5 : 2
    ctx.stroke()

    if (onPath) {
      ctx.beginPath()
      ctx.arc(s.x, s.y, 25, 0, Math.PI * 2)
      ctx.strokeStyle = 'rgba(34, 211, 238, 0.45)'
      ctx.lineWidth = 1.5
      ctx.stroke()
    }

    if (s.isAccept) {
      ctx.beginPath()
      ctx.arc(s.x, s.y, 15, 0, Math.PI * 2)
      ctx.strokeStyle = '#16a34a'
      ctx.lineWidth = 1.5
      ctx.stroke()
    }

    ctx.fillStyle = '#f1f5f9'
    ctx.font = 'bold 12px monospace'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(String(s.id), s.x, s.y)

    if (s.isStart) {
      ctx.beginPath()
      ctx.moveTo(s.x - 40, s.y)
      ctx.lineTo(s.x - 22, s.y)
      ctx.strokeStyle = '#06b6d4'
      ctx.lineWidth = 2
      ctx.stroke()
      ctx.beginPath()
      ctx.moveTo(s.x - 22, s.y)
      ctx.lineTo(s.x - 28, s.y - 4)
      ctx.lineTo(s.x - 28, s.y + 4)
      ctx.closePath()
      ctx.fillStyle = '#06b6d4'
      ctx.fill()
    }
  })

  ctx.restore()
  ctx.globalAlpha = 1
  drawMini()
}

// ---- 概览小地图：整体结构 + 当前视口位置 ----
function drawMini() {
  const mini = miniRef.value
  if (!mini) return
  const ctx = mini.getContext('2d')
  if (!ctx) return
  ctx.clearRect(0, 0, 160, 100)
  const nfa = store.nfa
  const t = miniTransform()
  if (!nfa || !t) return

  const byId = new Map(nfa.states.map(s => [s.id, s]))
  ctx.strokeStyle = 'rgba(100, 116, 139, 0.45)'
  ctx.lineWidth = 0.5
  nfa.transitions.forEach(tr => {
    const a = byId.get(tr.from), b = byId.get(tr.to)
    if (!a || !b) return
    ctx.beginPath()
    ctx.moveTo(a.x * t.s + t.ox, a.y * t.s + t.oy)
    ctx.lineTo(b.x * t.s + t.ox, b.y * t.s + t.oy)
    ctx.stroke()
  })

  const focus = focusInfo.value
  nfa.states.forEach(s => {
    const onPath = !!focus && focus.states.has(s.id)
    ctx.fillStyle = onPath ? '#22d3ee' : s.isStart ? '#06b6d4' : s.isAccept ? '#22c55e' : '#64748b'
    ctx.beginPath()
    ctx.arc(s.x * t.s + t.ox, s.y * t.s + t.oy, onPath ? 2.5 : 1.6, 0, Math.PI * 2)
    ctx.fill()
  })

  // 视口矩形
  const vx = (0 - view.x) / view.scale, vy = (0 - view.y) / view.scale
  const vw = canvasSize.w / view.scale, vh = canvasSize.h / view.scale
  ctx.strokeStyle = '#f59e0b'
  ctx.lineWidth = 1
  ctx.strokeRect(vx * t.s + t.ox, vy * t.s + t.oy, vw * t.s, vh * t.s)
}

// ---- 画布尺寸自适应 ----
function syncCanvasSize() {
  const canvas = canvasRef.value
  if (!canvas) return
  const dpr = window.devicePixelRatio || 1
  const w = canvas.parentElement ? canvas.parentElement.clientWidth : 800
  canvasSize.w = Math.max(280, w)
  canvasSize.h = 500
  canvas.width = Math.round(canvasSize.w * dpr)
  canvas.height = Math.round(canvasSize.h * dpr)
  canvas.style.height = canvasSize.h + 'px'
}

let resizeObs: ResizeObserver | null = null
onMounted(() => {
  syncCanvasSize()
  resetView()
  resizeObs = new ResizeObserver(() => { syncCanvasSize(); draw() })
  if (canvasRef.value?.parentElement) resizeObs.observe(canvasRef.value.parentElement)
})
onBeforeUnmount(() => resizeObs?.disconnect())

// 重新执行匹配（含模板套用）→ 恢复合理布局；切换步骤 → 仅重绘，保持视野
watch(() => store.nfa, () => resetView())
watch(() => [store.currentStep, store.matchResult, focusEnabled.value], () => draw())
</script>
