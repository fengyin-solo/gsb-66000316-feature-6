<template>
  <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
    <div class="flex items-center justify-between mb-3">
      <h3 class="text-sm font-bold text-slate-400">NFA 状态机可视化</h3>
      <div class="flex items-center gap-2">
        <span v-if="store.nfa" class="text-xs text-slate-500">{{ store.nfa.states.length }} 状态 · {{ store.nfa.transitions.length }} 转移</span>
        <button
          @click="focusMode = !focusMode"
          class="px-2 py-1 rounded text-xs border transition-colors"
          :class="focusMode ? 'bg-sky-900/60 border-sky-600 text-sky-300' : 'bg-slate-700 border-slate-600 text-slate-400 hover:text-slate-200'"
        >聚焦路径</button>
        <button
          @click="fitView"
          class="px-2 py-1 rounded text-xs bg-slate-700 border border-slate-600 text-slate-400 hover:text-slate-200"
        >重置视图</button>
      </div>
    </div>

    <div class="relative">
      <canvas
        ref="canvasRef"
        class="w-full h-[500px] bg-slate-900 rounded-lg border border-slate-700 select-none"
        :class="isPanning ? 'cursor-grabbing' : 'cursor-grab'"
        @mousedown="onMouseDown"
        @mousemove="onMouseMove"
        @mouseup="onMouseUp"
        @mouseleave="onMouseUp"
        @wheel.prevent="onWheel"
        @dblclick="fitView"
      ></canvas>

      <!-- 可读提示浮层 -->
      <div v-if="hints.length" class="absolute left-2 top-2 space-y-1 pointer-events-none max-w-[75%]">
        <div
          v-for="(h, i) in hints"
          :key="i"
          class="text-xs bg-slate-800/90 border border-amber-700/40 text-amber-300 px-2 py-1 rounded"
        >{{ h }}</div>
      </div>

      <!-- 缩放控制 -->
      <div class="absolute right-2 bottom-2 flex items-center gap-1">
        <button
          @click="zoomBy(0.8)"
          class="w-6 h-6 flex items-center justify-center bg-slate-800/90 hover:bg-slate-700 border border-slate-600 rounded text-slate-300 text-sm"
        >−</button>
        <span class="text-xs text-slate-400 bg-slate-800/90 border border-slate-600 px-1.5 py-0.5 rounded w-14 text-center">{{ zoomPercent }}%</span>
        <button
          @click="zoomBy(1.25)"
          class="w-6 h-6 flex items-center justify-center bg-slate-800/90 hover:bg-slate-700 border border-slate-600 rounded text-slate-300 text-sm"
        >＋</button>
      </div>
    </div>

    <div class="mt-2 flex flex-wrap gap-x-4 gap-y-1 text-xs text-slate-500">
      <span><span class="inline-block w-3 h-3 rounded-full bg-cyan-500 mr-1"></span>起始状态</span>
      <span><span class="inline-block w-3 h-3 rounded-full bg-green-500 mr-1"></span>接受状态</span>
      <span><span class="inline-block w-3 h-3 rounded-full bg-orange-500 mr-1"></span>当前激活</span>
      <span><span class="inline-block w-3 h-3 rounded-full bg-sky-600 mr-1"></span>聚焦路径</span>
      <span><span class="inline-block w-3 h-3 rounded-full bg-slate-600 mr-1"></span>普通状态</span>
      <span class="text-slate-600 ml-auto">拖拽平移 · 滚轮缩放 · 双击复位</span>
    </div>

    <!-- 路径看板 -->
    <div class="mt-3 bg-slate-900 rounded-lg border border-slate-700 p-3">
      <div class="flex items-center justify-between mb-2">
        <h4 class="text-xs font-bold text-slate-500">路径看板</h4>
        <span class="text-xs" :class="hasActivePath ? 'text-sky-400' : 'text-slate-600'">
          {{ hasActivePath ? '● 路径已激活' : '○ 暂无激活路径' }}
        </span>
      </div>
      <div class="grid grid-cols-2 sm:grid-cols-4 gap-2">
        <div class="bg-slate-800 rounded-lg p-2 text-center">
          <div class="text-xs text-slate-500 mb-0.5">当前步骤</div>
          <div class="text-sm font-bold text-cyan-400">
            {{ pathStats.current }}<span class="text-slate-500 font-normal"> / {{ pathStats.total }}</span>
          </div>
        </div>
        <div class="bg-slate-800 rounded-lg p-2 text-center">
          <div class="text-xs text-slate-500 mb-0.5">路径状态</div>
          <div class="text-sm font-bold text-sky-400">{{ pathStats.states }} 个</div>
        </div>
        <div class="bg-slate-800 rounded-lg p-2 text-center">
          <div class="text-xs text-slate-500 mb-0.5">路径转移</div>
          <div class="text-sm font-bold text-sky-400">{{ pathStats.edges }} 条</div>
        </div>
        <div class="bg-slate-800 rounded-lg p-2 text-center">
          <div class="text-xs text-slate-500 mb-0.5">路径覆盖率</div>
          <div class="text-sm font-bold text-emerald-400">{{ pathStats.coverage }}%</div>
          <div class="mt-1 h-1 bg-slate-700 rounded-full overflow-hidden">
            <div class="h-full bg-emerald-500 transition-all duration-200" :style="{ width: pathStats.coverage + '%' }"></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount, watch } from 'vue'
import { useRegexStore } from '../store/regex'
import type { NFA } from '../types'

const store = useRegexStore()
const canvasRef = ref<HTMLCanvasElement | null>(null)

// 聚焦路径模式：弱化与当前路径无关的节点和连线
const focusMode = ref(true)
const isPanning = ref(false)

// 视野变换：screen = world * scale + offset
const view = ref({ scale: 1, offsetX: 0, offsetY: 0 })

let cssWidth = 800
let cssHeight = 500
let dpr = 1
let resizeObserver: ResizeObserver | null = null
let panStart = { x: 0, y: 0, offsetX: 0, offsetY: 0 }

// 已走过的路径（第 0..currentStep 步涉及的状态与转移）
const pathInfo = computed(() => {
  const states = new Set<number>()
  const edges = new Set<string>()
  const steps = store.matchResult?.steps
  if (steps && steps.length) {
    const n = Math.min(store.currentStep, steps.length - 1)
    for (let i = 0; i <= n; i++) {
      const st = steps[i]
      if (st.currentState >= 0) states.add(st.currentState)
      if (st.nextState >= 0) states.add(st.nextState)
      if (st.currentState >= 0 && st.nextState >= 0) edges.add(st.currentState + '->' + st.nextState)
    }
  }
  return { states, edges }
})

const hasActivePath = computed(() => pathInfo.value.states.size > 0)

// 当前步骤的激活状态与转移
const activeStates = computed(() => {
  const set = new Set<number>()
  const steps = store.matchResult?.steps
  if (steps && steps.length) {
    const st = steps[Math.min(store.currentStep, steps.length - 1)]
    if (st) {
      if (st.currentState >= 0) set.add(st.currentState)
      if (st.nextState >= 0) set.add(st.nextState)
    }
  }
  return set
})

const currentEdge = computed(() => {
  const steps = store.matchResult?.steps
  if (!steps || !steps.length) return null
  const st = steps[Math.min(store.currentStep, steps.length - 1)]
  if (!st || st.currentState < 0 || st.nextState < 0) return null
  return { from: st.currentState, to: st.nextState }
})

// 节点重叠检测（间距小于节点直径）
const overlapCount = computed(() => {
  const states = store.nfa?.states ?? []
  let count = 0
  for (let i = 0; i < states.length; i++) {
    for (let j = i + 1; j < states.length; j++) {
      const dx = states[i].x - states[j].x
      const dy = states[i].y - states[j].y
      if (dx * dx + dy * dy < 44 * 44) count++
    }
  }
  return count
})

// 可读提示
const hints = computed(() => {
  const list: string[] = []
  const nfa = store.nfa
  if (!nfa) {
    list.push(store.error ? '解析失败，请修正正则后重新执行' : '尚未生成状态机，请先执行匹配')
    return list
  }
  const stateCount = nfa.states.length
  const transCount = nfa.transitions.length
  if (stateCount > 40 || transCount > 80) {
    list.push(`状态机规模较大（${stateCount} 状态 / ${transCount} 转移），可缩放或平移查看局部结构`)
  }
  if (overlapCount.value > 0) {
    list.push(`检测到 ${overlapCount.value} 处节点重叠，放大画布可分辨细节`)
  }
  const mr = store.matchResult
  if (!mr) {
    list.push('尚未执行匹配，当前仅展示状态机结构')
  } else if (mr.steps.length === 0) {
    list.push('没有可展示的匹配步骤，当前无激活路径')
  } else if (!hasActivePath.value) {
    list.push('当前步骤无激活路径')
  } else if (!mr.matched) {
    list.push('未匹配成功，路径展示至失败位置')
  }
  return list
})

// 路径统计（与当前步骤、匹配高亮同源，保持一致）
const pathStats = computed(() => {
  const total = store.matchResult?.steps.length ?? 0
  const stateCount = store.nfa?.states.length ?? 0
  return {
    current: total ? Math.min(store.currentStep, total - 1) : 0,
    total,
    states: pathInfo.value.states.size,
    edges: pathInfo.value.edges.size,
    coverage: stateCount ? Math.round((pathInfo.value.states.size / stateCount) * 100) : 0
  }
})

const zoomPercent = computed(() => Math.round(view.value.scale * 100))

function roundRectPath(ctx: CanvasRenderingContext2D, x: number, y: number, w: number, h: number, r: number) {
  ctx.beginPath()
  ctx.moveTo(x + r, y)
  ctx.arcTo(x + w, y, x + w, y + h, r)
  ctx.arcTo(x + w, y + h, x, y + h, r)
  ctx.arcTo(x, y + h, x, y, r)
  ctx.arcTo(x, y, x + w, y, r)
  ctx.closePath()
}

function draw() {
  const canvas = canvasRef.value
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  if (!ctx) return

  ctx.setTransform(dpr, 0, 0, 0, 0, 0)
  ctx.clearRect(0, 0, cssWidth, cssHeight)

  const nfa = store.nfa
  if (!nfa) {
    ctx.fillStyle = '#64748b'
    ctx.font = '13px sans-serif'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText('暂无状态机，请先执行匹配', cssWidth / 2, cssHeight / 2)
    return
  }

  const focus = focusMode.value && hasActivePath.value
  const path = pathInfo.value
  const active = activeStates.value
  const cur = currentEdge.value

  ctx.save()
  ctx.translate(view.value.offsetX, view.value.offsetY)
  ctx.scale(view.value.scale, view.value.scale)

  // 绘制转移
  nfa.transitions.forEach(t => {
    const from = nfa.states.find(s => s.id === t.from)
    const to = nfa.states.find(s => s.id === t.to)
    if (!from || !to) return

    const onPath = path.edges.has(t.from + '->' + t.to)
    const isCurrent = !!cur && cur.from === t.from && cur.to === t.to
    const dimmed = focus && !onPath && !isCurrent
    const highlightPath = focusMode.value && onPath

    ctx.save()
    if (dimmed) ctx.globalAlpha = 0.12

    const color = isCurrent ? '#f97316' : highlightPath ? '#38bdf8' : '#475569'
    ctx.strokeStyle = color
    ctx.lineWidth = isCurrent ? 3 : highlightPath ? 2 : 1
    ctx.beginPath()
    ctx.moveTo(from.x, from.y)

    if (t.from === t.to) {
      // 自环
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

    // 箭头
    let angle: number, ex: number, ey: number
    if (t.from === t.to) {
      angle = Math.atan2(20, -15)
      ex = from.x + 10
      ey = from.y - 15
    } else {
      angle = Math.atan2(to.y - from.y, to.x - from.x)
      ex = to.x - Math.cos(angle) * 22
      ey = to.y - Math.sin(angle) * 22
    }
    ctx.beginPath()
    ctx.moveTo(ex, ey)
    ctx.lineTo(ex - Math.cos(angle - 0.4) * 8, ey - Math.sin(angle - 0.4) * 8)
    ctx.lineTo(ex - Math.cos(angle + 0.4) * 8, ey - Math.sin(angle + 0.4) * 8)
    ctx.closePath()
    ctx.fillStyle = color
    ctx.fill()

    // 标签
    let lx: number, ly: number
    if (t.from === t.to) {
      lx = from.x
      ly = from.y - 46
    } else {
      const mx = (from.x + to.x) / 2, my = (from.y + to.y) / 2
      const dx = to.x - from.x, dy = to.y - from.y
      const len = Math.sqrt(dx * dx + dy * dy) || 1
      lx = mx - dy / len * 15
      ly = my + dx / len * 15
    }
    ctx.fillStyle = isCurrent ? '#fbbf24' : highlightPath ? '#7dd3fc' : '#94a3b8'
    ctx.font = '11px monospace'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(t.label, lx, ly)

    ctx.restore()
  })

  // 绘制状态
  nfa.states.forEach(s => {
    const isActive = active.has(s.id)
    const onPath = path.states.has(s.id)
    const dimmed = focus && !onPath && !isActive

    ctx.save()
    if (dimmed) ctx.globalAlpha = 0.15

    const color = s.isStart ? '#06b6d4' : s.isAccept ? '#22c55e' : isActive ? '#f97316' : focusMode.value && onPath ? '#0369a1' : '#475569'

    ctx.beginPath()
    ctx.arc(s.x, s.y, 20, 0, Math.PI * 2)
    ctx.fillStyle = color
    ctx.fill()
    ctx.strokeStyle = isActive ? '#fbbf24' : focusMode.value && onPath ? '#38bdf8' : '#1e293b'
    ctx.lineWidth = 2
    ctx.stroke()

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

    ctx.restore()
  })

  ctx.restore()

  drawMinimap(ctx, nfa)
}

// 概览（小地图）：整体结构 + 当前视野框
function drawMinimap(ctx: CanvasRenderingContext2D, nfa: NFA) {
  if (!nfa.states.length) return
  const mw = 150, mh = 100
  const mx = cssWidth - mw - 10, my = 10

  ctx.save()
  ctx.fillStyle = 'rgba(2, 6, 23, 0.8)'
  ctx.strokeStyle = '#334155'
  ctx.lineWidth = 1
  roundRectPath(ctx, mx, my, mw, mh, 6)
  ctx.fill()
  ctx.stroke()

  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity
  nfa.states.forEach(s => {
    minX = Math.min(minX, s.x); maxX = Math.max(maxX, s.x)
    minY = Math.min(minY, s.y); maxY = Math.max(maxY, s.y)
  })
  const pad = 30
  minX -= pad; minY -= pad; maxX += pad; maxY += pad
  const bw = Math.max(1, maxX - minX), bh = Math.max(1, maxY - minY)
  const k = Math.min((mw - 16) / bw, (mh - 16) / bh)
  const ox = mx + (mw - bw * k) / 2, oy = my + (mh - bh * k) / 2
  const px = (x: number) => ox + (x - minX) * k
  const py = (y: number) => oy + (y - minY) * k

  // 转移
  ctx.strokeStyle = 'rgba(100, 116, 139, 0.5)'
  ctx.lineWidth = 0.5
  nfa.transitions.forEach(t => {
    const from = nfa.states.find(s => s.id === t.from)
    const to = nfa.states.find(s => s.id === t.to)
    if (!from || !to) return
    ctx.beginPath()
    ctx.moveTo(px(from.x), py(from.y))
    ctx.lineTo(px(to.x), py(to.y))
    ctx.stroke()
  })

  // 状态
  const path = pathInfo.value
  nfa.states.forEach(s => {
    ctx.beginPath()
    ctx.arc(px(s.x), py(s.y), 2, 0, Math.PI * 2)
    ctx.fillStyle = s.isStart ? '#06b6d4' : s.isAccept ? '#22c55e' : focusMode.value && path.states.has(s.id) ? '#38bdf8' : '#64748b'
    ctx.fill()
  })

  // 当前视野框
  const vx = -view.value.offsetX / view.value.scale
  const vy = -view.value.offsetY / view.value.scale
  const vw = cssWidth / view.value.scale
  const vh = cssHeight / view.value.scale
  ctx.strokeStyle = '#22d3ee'
  ctx.lineWidth = 1
  ctx.strokeRect(px(vx), py(vy), vw * k, vh * k)

  ctx.fillStyle = '#475569'
  ctx.font = '9px sans-serif'
  ctx.textAlign = 'left'
  ctx.textBaseline = 'top'
  ctx.fillText('概览', mx + 6, my + 4)
  ctx.restore()
}

// 视野适配：重新执行匹配 / 套用模板后恢复合理布局
function fitView() {
  const nfa = store.nfa
  if (!nfa || !nfa.states.length) {
    view.value = { scale: 1, offsetX: 0, offsetY: 0 }
    draw()
    return
  }
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity
  nfa.states.forEach(s => {
    minX = Math.min(minX, s.x); maxX = Math.max(maxX, s.x)
    minY = Math.min(minY, s.y); maxY = Math.max(maxY, s.y)
  })
  const pad = 50
  const bw = Math.max(1, maxX - minX + pad * 2)
  const bh = Math.max(1, maxY - minY + pad * 2)
  const scale = Math.min(cssWidth / bw, cssHeight / bh, 1.4)
  view.value = {
    scale,
    offsetX: (cssWidth - (maxX - minX) * scale) / 2 - minX * scale,
    offsetY: (cssHeight - (maxY - minY) * scale) / 2 - minY * scale
  }
  draw()
}

function zoomBy(factor: number, cx?: number, cy?: number) {
  const mx = cx ?? cssWidth / 2
  const my = cy ?? cssHeight / 2
  const newScale = Math.min(4, Math.max(0.1, view.value.scale * factor))
  // 以 (mx, my) 为锚点缩放
  const wx = (mx - view.value.offsetX) / view.value.scale
  const wy = (my - view.value.offsetY) / view.value.scale
  view.value = {
    scale: newScale,
    offsetX: mx - wx * newScale,
    offsetY: my - wy * newScale
  }
  draw()
}

function onMouseDown(e: MouseEvent) {
  isPanning.value = true
  panStart = { x: e.clientX, y: e.clientY, offsetX: view.value.offsetX, offsetY: view.value.offsetY }
}

function onMouseMove(e: MouseEvent) {
  if (!isPanning.value) return
  view.value = {
    scale: view.value.scale,
    offsetX: panStart.offsetX + e.clientX - panStart.x,
    offsetY: panStart.offsetY + e.clientY - panStart.y
  }
  draw()
}

function onMouseUp() {
  isPanning.value = false
}

function onWheel(e: WheelEvent) {
  const canvas = canvasRef.value
  if (!canvas) return
  const rect = canvas.getBoundingClientRect()
  zoomBy(Math.exp(-e.deltaY * 0.0012), e.clientX - rect.left, e.clientY - rect.top)
}

function resizeCanvas() {
  const canvas = canvasRef.value
  if (!canvas) return
  const rect = canvas.getBoundingClientRect()
  if (rect.width < 10 || rect.height < 10) return
  dpr = window.devicePixelRatio || 1
  cssWidth = rect.width
  cssHeight = rect.height
  canvas.width = Math.round(rect.width * dpr)
  canvas.height = Math.round(rect.height * dpr)
}

onMounted(() => {
  resizeCanvas()
  fitView()
  if (canvasRef.value) {
    resizeObserver = new ResizeObserver(() => {
      resizeCanvas()
      draw()
    })
    resizeObserver.observe(canvasRef.value)
  }
})

onBeforeUnmount(() => {
  resizeObserver?.disconnect()
})

// 重新执行匹配 / 套用模板 → nfa 引用变化 → 恢复合理布局
watch(() => store.nfa, () => fitView())
// 切换步骤 / 匹配结果变化 → 只重绘，保持当前平移缩放视野
watch(() => [store.currentStep, store.matchResult], () => draw(), { deep: true })
watch(focusMode, () => draw())
</script>
