<!-- 原本 所有 CSV 文件在 onMounted 中预加载到 csvData.value，而 chartData 的 computed 属性会根据 selectedCsv.value 重新计算图表数据 的版本 -->
<!-- 后来改成了 使用 watch 监听 CSV 切换的机制 -->

<template>
  <div v-if="errorMessage" class="error" role="alert">{{ errorMessage }}</div>
  <div v-else-if="!ready">Loading...</div>
  <div v-else class="wrapper">
    <!-- ① CSV文件选择区域 -->
    <div class="filters">
      <fieldset>
        <legend>RNA modifications</legend>
        <!-- 使用单选按钮选择单个CSV文件，显示时去掉 .csv 后缀 -->
        <label v-for="csv in csvFiles" :key="csv" class="checkbox">
          <input type="radio" :value="csv" v-model="selectedCsv" name="csv-selection" />
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
          {{ isAllModelsSelected ? 'Clear' : 'All' }}
        </button>

        <!-- ✅ 动态复选框 -->
        <label v-for="name in modelNames" :key="name" class="checkbox">
          <input type="checkbox" :value="name" v-model="selectedModels" />
          {{ name }}
        </label>
      </fieldset>
    </div>

    <!-- ③ 雷达图 -->
    <div class="chart-box">
      <Radar :data="chartData" :options="chartOptions" ref="chartRef" />
    </div>
  </div>
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

// function csvToJson(csv) {
//   const lines = csv.trim().split('\n')
//   // const _lbls = lines[0].split(',').slice(1).map(label => label.length > 15 ? label.slice(0, 12) + '...' : label)
//   const _lbls = lines[0].split(',').slice(1)
//   const modelData = {} // 避免与 chartData 中的变量冲突
//   for (let i = 1; i < lines.length; i++) {
//     const cols = lines[i].split(',')
//     modelData[cols[0]] = cols.slice(1).map(Number)
//   }
//   return { labels: _lbls, data: modelData }
// }


function csvToJson(csv) {
  try {
    const lines = csv.trim().split('\n')
    if (lines.length < 2) throw new Error('CSV file is empty or invalid')
    const _lbls = lines[0].split(',').slice(1).map(label => label.trim())
    if (!_lbls.length) throw new Error('No labels found in CSV')
    const modelData = {}
    for (let i = 1; i < lines.length; i++) {
      const cols = lines[i].split(',')
      if (cols.length !== _lbls.length + 1) throw new Error(`Invalid row format at line ${i + 1}`)
      const values = cols.slice(1).map(val => {
        const num = Number(val)
        // if (isNaN(num)) throw new Error(`Non-numeric value "${val}" in row ${i + 1}`)
        return num
      })
      modelData[cols[0].trim()] = values
    }
    return { labels: _lbls, data: modelData }
  } catch (error) {
    throw new Error(`CSV parsing error: ${error.message}`)
  }
}

/* ---------- 3. 标签到图片的映射 -------- */
const labelImageMap = computed(() => {
  // Map labels to image paths (adjust based on your CSV labels)
  const map = {}
  labels.value.forEach(label => {
    // Use lowercase and remove spaces for image filenames
    const imageName = label.toLowerCase().replace(/\s+/g, '') + '.png'
    // map[label] = `public/icons/${imageName}` // Images at /icons/<label>.png
    map[label] = `/icons/${imageName}` // public 好像不用写? 但是不加上又无法显示
  })
  return map
})

/* ---------- 4. 加载 CSV 文件 -------- */
onMounted(async () => {
  try {
    // 使用 Vite 的 import.meta.glob 动态加载 CSV 文件
    console.log('Starting onMounted')
    const modules = import.meta.glob('/public/*.csv', { as: 'raw', eager: true })
    csvFiles.value = Object.keys(modules).map(file => file.split('/').pop())
    selectedCsv.value = csvFiles.value[0] || null // 默认选中第一个文件
    console.log('CSV files:', csvFiles.value)

    if (!selectedCsv.value) {
      // console.error('No CSV files found in public/')
      // return
      throw new Error('No CSV files found in public/') // add 2025-07-12 17:38:25
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
    // selectedModels.value = []
    // ready.value = true
    // 默认全选 2025-07-12 17:40:16
    selectedModels.value = modelNames.value

    console.log('Data loaded for', initialCsv, { labels: _lbls, datasets: entries })

    // Preload images for labels
    Object.values(labelImageMap.value).forEach(src => {
      const img = new Image()
      img.src = src
    })
    console.log('labelImageMap.value: ', labelImageMap.value)
    console.log('Label image map:', labelImageMap.value)
    //  默认全选 2025-07-12 17:40:20
    ready.value = true

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
    errorMessage.value = `Initialization failed: ${error.message}` // 2025-07-12 17:41:58
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

/* ---------- 5. 计算图表数据（随CSV和模型选择变化） -------- */
const chartData = computed(() => {
  if (!selectedCsv.value || !csvData.value[selectedCsv.value]) {
    console.warn('No data for chartData')
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

/* ---------- 6. 图表配置 -------- */
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
    },
    // Custom plugin to draw images next to point labels
    pointLabelImages: {
      id: 'pointLabelImages',
      afterDraw(chart) {
        const { ctx, scales: { r } } = chart
        const labels = chart.data.labels
        const labelPositions = r._pointLabelItems

        if (!labelPositions) return

        labels.forEach((label, index) => {
          const imageSrc = labelImageMap.value[label]
          if (!imageSrc) return // Skip if no image for label

          const { x, y } = labelPositions[index]
          const img = new Image()
          img.src = imageSrc
          if (img.complete) {
            // Draw image to the left of the label
            const imageSize = 150 // Size in pixels
            const offset = 25 // Distance from label
            ctx.drawImage(img, x - offset - imageSize, y - imageSize / 2, imageSize, imageSize)
          } else {
            img.onload = () => {
              ctx.drawImage(img, x - offset - imageSize, y - imageSize / 2, imageSize, imageSize)
              chart.update() // Redraw chart to ensure image appears
            }
          }
        })
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

/* ---------- 7. 模型全选/取消全选逻辑 -------- */
const isAllModelsSelected = computed(() =>
  selectedModels.value.length === modelNames.value.length
)

function toggleAllModels() {
  selectedModels.value = isAllModelsSelected.value ? [] : [...modelNames.value]
}

/* ---------- 8. 注册自定义插件 -------- */
ChartJS.register({
  id: 'pointLabelImages',
  afterDraw: chartOptions.plugins.pointLabelImages.afterDraw
  // }, {
  //   id: 'lineHoverPlugin',
  //   beforeEvent: chartOptions.plugins.lineHoverPlugin.beforeEvent
  // //加上这段使得悬停无法高亮, 悬停并点击才能高亮
})
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

.error {
  color: red;
  padding: 1rem;
  text-align: center;
}
</style>
