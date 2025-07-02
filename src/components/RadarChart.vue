
<template>
  <div v-if="ready" class="wrapper">
    <!-- ① CSV文件选择区域 -->
    <div class="filters">
      <fieldset>
        <legend>RNA modifications</legend>
        <!-- 使用单选按钮选择单个CSV文件，显示时去掉 .csv 后缀 -->
        <label v-for="csv in csvFiles" :key="csv" class="checkbox">
          <input
            type="radio"
            :value="csv"
            v-model="selectedCsv"
            name="csv-selection"
          />
          {{ csv.replace('.csv', '') }}
        </label>
      </fieldset>
    </div>

    <!-- ② 模型选择区域 -->
    <div class="filters">
      <fieldset>
        <legend>Models</legend>
        <!-- 🔘 一键切换按钮 -->
        <button @click="toggleAllModels" class="toggle-btn">
          {{ isAllModelsSelected ? 'None' : 'All' }}
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

    <!-- ③ 雷达图 -->
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
const csvFiles = ref([]) // 可用的CSV文件列表，动态获取
const selectedCsv = ref(null) // 当前选中的CSV文件，初始为 null
const csvData = ref({}) // 缓存所有CSV文件的数据
const labels = ref([]) // 当前选中的CSV文件的指标
const datasets = ref([]) // 当前选中的CSV文件的模型数据集
const modelNames = ref([]) // 当前选中的CSV文件的模型名称数组
const selectedModels = ref([]) // 当前选中的模型
const hoveredDatasetIndex = ref(null) // 当前高亮的 dataset 索引
const hoveredDataIndex = ref(null) // 当前高亮的数据点索引
const chartRef = ref(null) // 引用 Radar 组件

/* ---------- 2. 工具函数 -------- */
function generateColors(n) {
  return Array.from({ length: n }, (_, i) =>
    `hsl(${Math.round((360 / n) * i)}, 70%, 60%)`
  )
}

function csvToJson(csv) {
  const lines = csv.trim().split('\n')
  // const _lbls = lines[0].split(',').slice(1).map(label => label.length > 15 ? label.slice(0, 12) + '...' : label)
  const _lbls = lines[0].split(',').slice(1)
  const modelData = {} // 避免与 chartData 中的变量冲突
  for (let i = 1; i < lines.length; i++) {
    const cols = lines[i].split(',')
    modelData[cols[0]] = cols.slice(1).map(Number)
  }
  return { labels: _lbls, data: modelData }
}

/* ---------- 3. 加载 CSV 文件 -------- */
onMounted(async () => {
  try {
    // 使用 Vite 的 import.meta.glob 动态加载 CSV 文件
    const modules = import.meta.glob('/public/*.csv', { as: 'raw', eager: true })
    csvFiles.value = Object.keys(modules).map(file => file.split('/').pop())
    selectedCsv.value = csvFiles.value[0] || null // 默认选中第一个文件

    if (!selectedCsv.value) {
      console.error('No CSV files found in src/assets/')
      return
    }

    // 缓存所有CSV文件数据
    for (const file of csvFiles.value) {
      csvData.value[file] = csvToJson(modules[`/public/${file}`])
    }

    // 初始化第一个CSV文件的数据
    const initialCsv = selectedCsv.value
    const { labels: _lbls, data: modelData } = csvData.value[initialCsv] || { labels: [], data: {} }
    const entries = Object.entries(modelData).filter(([name]) => !name.includes('Max') && !name.includes('Min'))
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
      pointHitRadius: 10,
      hoverBorderWidth: 4,
      originalBorderColor: colors[i],
      originalBackgroundColor: colors[i].replace('hsl', 'hsla').replace(')', ', 0.2)')
    }))
    modelNames.value = entries.map(([name]) => name)
    selectedModels.value = []
    ready.value = true
    console.log('Data loaded for', initialCsv, { labels: _lbls, datasets: entries })

    // 添加 mouseleave 事件监听器
    const canvas = chartRef.value?.chart?.canvas
    if (canvas) {
      canvas.addEventListener('mouseleave', () => {
        console.log('Mouse left chart')
        hoveredDatasetIndex.value = null
        hoveredDataIndex.value = null
      })
    } else {
      console.error('Canvas not found')
    }
  } catch (error) {
    console.error('Error loading CSVs:', error)
  }
})

/* 清理事件监听器 */
onUnmounted(() => {
  const canvas = chartRef.value?.chart?.canvas
  if (canvas) {
    canvas.removeEventListener('mouseleave', () => {
      hoveredDatasetIndex.value = null
      hoveredDataIndex.value = null
    })
  }
})

/* ---------- 4. 计算图表数据（随CSV和模型选择变化） -------- */
const chartData = computed(() => {
  if (!selectedCsv.value || !csvData.value[selectedCsv.value]) {
    return { labels: [], datasets: [] }
  }

  // 获取当前选中的CSV文件数据
  const { labels: _lbls, data: modelData } = csvData.value[selectedCsv.value]
  const entries = Object.entries(modelData).filter(([name]) => !name.includes('Max') && !name.includes('Min'))
  const colors = generateColors(entries.length)

  // 更新 labels 和 datasets
  labels.value = _lbls
  datasets.value = entries.map(([name, values], i) => ({
    label: name,
    data: values,
    borderColor: colors[i],
    backgroundColor: colors[i].replace('hsl', 'hsla').replace(')', ', 0.2)'),
    pointBackgroundColor: colors[i],
    borderWidth: 2,
    fill: true,
    pointHitRadius: 10,
    hoverBorderWidth: 4,
    originalBorderColor: colors[i],
    originalBackgroundColor: colors[i].replace('hsl', 'hsla').replace(')', ', 0.2)')
  }))
  modelNames.value = entries.map(([name]) => name)

  // 过滤选中的模型
  const selectedDatasets = datasets.value.filter(ds => selectedModels.value.includes(ds.label)).map((ds, i) => {
    const isHovered = hoveredDatasetIndex.value === i
    return {
      ...ds,
      borderColor: isHovered ? ds.originalBorderColor : ds.originalBorderColor.replace('hsl', 'hsla').replace(')', ', 0.3)'),
      backgroundColor: isHovered ? ds.originalBackgroundColor : ds.originalBackgroundColor.replace('0.2)', '0)'),
      borderWidth: isHovered ? 4 : 2
    }
  })

  const chartDataObject = {
    labels: labels.value,
    datasets: selectedDatasets
  }
  console.log('Chart data:', chartDataObject)
  return chartDataObject
})

/* ---------- 5. 图表配置 -------- */
const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  layout: {
    padding: 40 // 防止标签被裁剪
  },
  interaction: {
    mode: 'point', // 仅检测单个数据点
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
      enabled: true,
      position: 'nearest', // 使用默认定位器
      backgroundColor: 'rgba(0, 0, 0, 0.7)', // 半透明背景
      bodyFont: { size: 12 },
      padding: 8,
      callbacks: {
        title: () => '', // 移除标题
        label: ctx => {
          // 只显示当前悬停的数据点
          if (ctx.datasetIndex === hoveredDatasetIndex.value && ctx.dataIndex === hoveredDataIndex.value) {
            const value = Number(ctx.raw).toFixed(4) // 四舍五入到4位小数
            const label = `${ctx.chart.data.labels[ctx.dataIndex]}: ${value}`
            console.log('Tooltip:', label)
            return label
          }
          return ''
        }
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
          hoveredDataIndex.value = null // 无数据点索引，因为线条悬停不触发tooltip
          console.log('Line hover dataset:', closestDatasetIndex)
        }
      }
    }
  },
  onHover: (event, chartElements) => {
    console.log('Hover elements:', chartElements)
    if (chartElements.length > 0) {
      const datasetIndex = chartElements[0].datasetIndex
      const dataIndex = chartElements[0].index
      hoveredDatasetIndex.value = datasetIndex
      hoveredDataIndex.value = dataIndex
    } else if (hoveredDataIndex.value !== null) {
      hoveredDatasetIndex.value = null
      hoveredDataIndex.value = null
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

/* ---------- 6. 模型全选/取消全选逻辑 -------- */
const isAllModelsSelected = computed(() =>
  selectedModels.value.length === modelNames.value.length
)

function toggleAllModels() {
  selectedModels.value = isAllModelsSelected.value ? [] : [...modelNames.value]
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
  min-width: 60vw;
  position: relative;
}
.chart-box canvas {
  width: 100% !important;
  height: 100% !important;
}
</style>
