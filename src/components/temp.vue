
<template>
  <div v-if="ready" class="wrapper">
    <!-- ① 筛选区域 -->
    <div class="filters">
      <fieldset>
        <legend>选择要显示的模型</legend>
        <!-- 🔘 一键切换按钮 -->
        <button @click="toggleAll" class="toggle-btn">
          {{ isAllSelected ? '取消全选' : '全选所有' }}
        </button>

        <!-- ✅ 动态复选框 -->
        <label v-for="name in modelNames" :key="name" class="checkbox">
          <input
            type="checkbox"
            :value="name"
            v-model="selectedModels"
          />
          {{ name }}
        </label>
      </fieldset>
    </div>

    <!-- ② 雷达图 -->
    <div class="chart-box">
      <Radar :data="chartData" :options="chartOptions" ref="chartRef" />
    </div>
  </div>
  <div v-else>Loading...</div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { Radar } from 'vue-chartjs'
import {
  Chart as ChartJS,
  Title, Tooltip, Legend,
  RadialLinearScale, PointElement, LineElement, Filler
} from 'chart.js'

ChartJS.register(
  Title, Tooltip, Legend,
  RadialLinearScale, PointElement, LineElement, Filler
)

/* ---------- 1. 状态 --------- */
const ready = ref(false) // 数据加载完成标记
const labels = ref([]) // 指标
const datasets = ref([]) // 所有模型数据集
const modelNames = ref([]) // 模型名称数组
const selectedModels = ref([]) // 当前勾选的模型
const hoveredDatasetIndex = ref(null) // 当前高亮的 dataset 索引
const chartRef = ref(null) // 引用 Radar 组件

/* ---------- 2. 工具函数 -------- */
function generateColors(n) {
  return Array.from({ length: n }, (_, i) =>
    `hsl(${Math.round((360 / n) * i)}, 70%, 60%)`
  )
}

function csvToJson(csv) {
  const lines = csv.trim().split('\n')
  const _lbls = lines[0].split(',').slice(1).map(label => label.length > 15 ? label.slice(0, 12) + '...' : label)
  const data = {}
  for (let i = 1; i < lines.length; i++) {
    const cols = lines[i].split(',')
    data[cols[0]] = cols.slice(1).map(Number)
  }
  return { labels: _lbls, data }
}

/* ---------- 3. 加载 CSV -------- */
onMounted(async () => {
  const res = await fetch('/data.csv')
  const text = await res.text()
  const { labels: _lbls, data } = csvToJson(text)

  const entries = Object.entries(data).filter(([name]) => name !== 'Max' && name !== 'Min')

  /* 自动配色 */
  const colors = generateColors(entries.length)

  labels.value = _lbls
  datasets.value = entries.map(([name, values], i) => ({
    label: name,
    data: values,
    borderColor: colors[i],
    backgroundColor: colors[i].replace('hsl', 'hsla').replace(')', ', 0.2)'),
    pointBackgroundColor: colors[i],
    borderWidth: 2,
    fill: true,
    pointHitRadius: 10, // 增大点检测范围
    hoverBorderWidth: 4, // 加粗悬停时的线条
    originalBorderColor: colors[i],
    originalBackgroundColor: colors[i].replace('hsl', 'hsla').replace(')', ', 0.2)')
  }))
  modelNames.value = entries.map(([name]) => name)
  selectedModels.value = []
  ready.value = true

  // 添加 mouseleave 事件监听器
  const canvas = chartRef.value?.chart?.canvas
  if (canvas) {
    canvas.addEventListener('mouseleave', () => {
      hoveredDatasetIndex.value = null
    })
  }
})

/* 清理事件监听器 */
onUnmounted(() => {
  const canvas = chartRef.value?.chart?.canvas
  if (canvas) {
    canvas.removeEventListener('mouseleave', () => {
      hoveredDatasetIndex.value = null
    })
  }
})

/* ---------- 4. 计算图表数据（随勾选和悬停变化） -------- */
const chartData = computed(() => ({
  labels: labels.value,
  datasets: datasets.value
    .filter(ds => selectedModels.value.includes(ds.label))
    .map((ds, i) => {
      const isHovered = hoveredDatasetIndex.value === i
      return {
        ...ds,
        borderColor: isHovered ? ds.originalBorderColor : ds.originalBorderColor.replace('hsl', 'hsla').replace(')', ', 0.3)'),
        backgroundColor: isHovered ? ds.originalBackgroundColor : ds.originalBackgroundColor.replace('0.2)', '0.05)'),
        borderWidth: isHovered ? 4 : 2
      }
    })
}))

/* ---------- 5. 图表配置 -------- */
const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  layout: {
    padding: 40 // 防止标签被裁剪
  },
  interaction: {
    mode: 'dataset', // 检测整个 dataset（点、线、填充区域）
    intersect: true,
    includeInvisible: false
  },
  scales: {
    r: {
      beginAtZero: true,
      max: 1,
      grid: { circular: true },
      pointLabels: {
        font: { size: 14 },
        padding: 15
      },
      ticks: {
        stepSize: 0.2,
        font: { size: 12 }
      }
    }
  },
  plugins: {
    legend: { display: false },
    tooltip: {
      callbacks: {
        label: ctx => `${ctx.chart.data.labels[ctx.dataIndex]}: ${ctx.raw}` // 使用指标名称
      }
    },
    lineHoverPlugin: {
      id: 'lineHoverPlugin',
      beforeEvent(chart, args) {
        const event = args.event
        if (event.type === 'mousemove') {
          const { ctx, chartArea } = chart
          const datasets = chart.data.datasets
          let closestDatasetIndex = null
          let minDistance = Infinity

          datasets.forEach((dataset, datasetIndex) => {
            if (!chart.getDatasetMeta(datasetIndex).visible) return

            const meta = chart.getDatasetMeta(datasetIndex)
            const elements = meta.data
            const points = elements.map((el, i) => ({
              x: el.x,
              y: el.y,
              value: dataset.data[i]
            }))

            for (let i = 0; i < points.length - 1; i++) {
              const p1 = points[i]
              const p2 = points[i + 1] || points[0]
              const distance = pointToLineDistance(event.x, event.y, p1.x, p1.y, p2.x, p2.y)
              if (distance < minDistance && distance < 10) {
                minDistance = distance
                closestDatasetIndex = datasetIndex
              }
            }
          })

          hoveredDatasetIndex.value = closestDatasetIndex
        }
      }
    }
  },
  onHover: (event, chartElements) => {
    if (chartElements.length > 0) {
      const datasetIndex = chartElements[0].datasetIndex
      hoveredDatasetIndex.value = datasetIndex
    } else if (hoveredDatasetIndex.value !== null) {
      // 仅当插件未检测到线条时清空
      hoveredDatasetIndex.value = null
    }
  }
}

// 计算点到线段的最短距离
function pointToLineDistance(px, py, x1, y1, x2, y2) {
  const dx = x2 - x1
  const dy = y2 - y1
  const lenSquared = dx * dx + dy * dy
  if (lenSquared === 0) return Math.sqrt((px - x1) ** 2 + (py - y1) ** 2)

  const t = Math.max(0, Math.min(1, ((px - x1) * dx + (py - y1) * dy) / lenSquared))
  const projX = x1 + t * dx
  const projY = y1 + t * dy
  return Math.sqrt((px - projX) ** 2 + (py - projY) ** 2)
}

/* ---------- 6. 全选 / 取消全选逻辑 -------- */
const isAllSelected = computed(() =>
  selectedModels.value.length === modelNames.value.length
)

function toggleAll() {
  selectedModels.value = isAllSelected.value ? [] : [...modelNames.value]
}
</script>

<style scoped>
.wrapper {
  display: flex;
  gap: 1.5rem;
  align-items: flex-start;
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
}
.filters {
  min-width: 200px;
  text-align: start;
}
.checkbox {
  display: block;
  margin: 0.2rem 0;
  cursor: pointer;
}
.toggle-btn {
  display: inline-block;
  margin-bottom: 0.5rem;
  padding: 0.25rem 0.5rem;
  font-size: 0.9rem;
  cursor: pointer;
  background-color: #f0f0f0;
  border: 1px solid #ccc;
  border-radius: 4px;
}
.chart-box {
  flex: 1 1 600px;
  min-height: 650px;
  min-width: 50vw;
  position: relative;
}
.chart-box canvas {
  width: 100% !important;
  height: 100% !important;
}
</style>
