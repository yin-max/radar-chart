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
      <Radar :data="chartData" :options="chartOptions" />
    </div>
  </div>
  <div v-else>Loading...</div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
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

/* ---------- 1. 状态变量 --------- */
const ready          = ref(false)
const labels         = ref([])
const datasets       = ref([])
const modelNames     = ref([])
const selectedModels = ref([])

/* ---------- 2. 工具函数 -------- */
function generateColors (n) {
  return Array.from({ length: n }, (_, i) =>
    `hsl(${Math.round((360 / n) * i)}, 70%, 60%)`
  )
}

function csvToJson (csv) {
  const lines  = csv.trim().split('\n')
  const _lbls  = lines[0].split(',').slice(1)
  const data   = {}
  for (let i = 1; i < lines.length; i++) {
    const cols = lines[i].split(',')
    data[cols[0]] = cols.slice(1).map(Number)
  }
  return { labels: _lbls, data }
}

/* ---------- 3. 加载 CSV -------- */
onMounted(async () => {
  const res  = await fetch('/data.csv')
  const text = await res.text()
  const { labels: _lbls, data } = csvToJson(text)

  const entries = Object.entries(data)
    .filter(([name]) => name !== 'Max' && name !== 'Min')

  const colors = generateColors(entries.length)

  labels.value   = _lbls
  datasets.value = entries.map(([name, values], i) => ({
    label: name,
    data : values,
    borderColor        : colors[i],
    backgroundColor    : colors[i] + '80',
    pointBackgroundColor: colors[i],
    borderWidth: 2,
    fill: true
  }))
  modelNames.value     = entries.map(([name]) => name)
  selectedModels.value = []   // ✅ 默认全不选
  ready.value          = true
})

/* ---------- 4. 图表数据 -------- */
const chartData = computed(() => ({
  labels: labels.value,
  datasets: datasets.value.filter(ds => selectedModels.value.includes(ds.label))
}))

/* ---------- 5. 图表配置 -------- */
const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  scales: {
    r: {
      beginAtZero: true,
      max: 1,
      grid: { circular: true },
      pointLabels: { font: { size: 12 } }
    }
  },
  plugins: {
    legend: { display: false },
    tooltip: {
      callbacks: {
        label: ctx => `${ctx.dataset.label}: ${ctx.formattedValue}`
      }
    }
  }
}

/* ---------- 6. 全选 / 取消全选逻辑 -------- */
const isAllSelected = computed(() =>
  selectedModels.value.length === modelNames.value.length
)

function toggleAll () {
  selectedModels.value = isAllSelected.value ? [] : [...modelNames.value]
}
</script>

<style scoped>
.wrapper    { display: flex; gap: 1.5rem; align-items: flex-start; }
.filters    { min-width: 200px; }
.checkbox   { display: block; margin: 0.2rem 0; cursor: pointer; }
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
.chart-box { flex: 1 1 600px; min-height: 650px; position: relative; }
</style>
