<template>
  <div v-if="errorMessage" class="error" role="alert">{{ errorMessage }}</div>
  <div v-else-if="!ready">Loading...</div>
  <div v-else class="container">
    <!-- ① 雷达图 -->
    <div class="wrapper">
      <div class="chart-box">
        <Radar :data="chartData" :options="chartOptions" ref="chartRef" />
      </div>
    </div>

    <!-- ② CSV文件选择区域和模型选择区域 -->
    <div class="filters-container">
      <div class="filters">
        <fieldset>
          <legend>RNA modifications</legend>
          <!-- 使用单选按钮选择单个CSV文件，显示时去掉 .csv 后缀 -->
          <label v-for="csv in csvFiles" :key="csv" class="checkbox">
            <input type="radio" :value="csv" v-model="selectedCsv" name="csv-selection" :aria-label="`Select CSV ${csv.replace('.csv', '')}`" />
            {{ csv.replace('.csv', '') }}
          </label>
        </fieldset>
      </div>

      <!-- ② 模型选择区域 -->
      <div class="filters">
        <fieldset>
          <!-- 🔘 一键切换按钮 -->
          <legend>
            Models
            <button @click="toggleAllModels" class="toggle-btn" :aria-label="isAllModelsSelected ? 'Clear all models' : 'Select all models'">
              {{ isAllModelsSelected ? 'Clear' : 'All' }}
            </button>
          </legend>
          <!-- ✅ 动态复选框 -->
          <div class="model-columns">
            <div v-for="(column, colIndex) in modelColumns" :key="colIndex" class="column">
              <label v-for="name in column" :key="name" class="checkbox" :class="{ selected: selectedModels.includes(name) }" :style="{ borderColor: selectedModels.includes(name) ? modelColors[name] || '#ccc' : 'transparent' }" :title="name">
              <!-- <label v-for="name in column" :key="name" class="checkbox" :class="{ selected: selectedModels.includes(name) }" :style="{ borderColor: selectedModels.includes(name) ? modelColors[name] || '#ccc' : 'transparent' }"> -->
              <!-- <label v-for="name in column" :key="name" class="checkbox"> -->
                <!-- <span class="color-indicator" :style="{ backgroundColor: modelColors[name] || '#ccc' }"></span> -->
                <input type="checkbox" :value="name" v-model="selectedModels" :aria-label="`Select model ${name}`" />
                <span class="model-name">{{ name }}</span>
              </label>
            </div>
          </div>
        </fieldset>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue';
import { Radar } from 'vue-chartjs';
import {
  Chart as ChartJS,
  Title, Tooltip, Legend,
  RadialLinearScale, PointElement, LineElement, Filler
} from 'chart.js';

ChartJS.register(
  Title, Tooltip, Legend,
  RadialLinearScale, PointElement, LineElement, Filler
);

/* ---------- 1. 状态 --------- */
const ready = ref(false) // 数据加载完成标记
const errorMessage = ref(null)
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
const imageCache = ref({}) // 图片缓存
const pointRadius = ref(2) // 默认点大小（像素）
const pointHoverRadius = ref(4) // 悬停时点大小
const borderWidth = ref(4) // 默认线条粗细
const hoverBorderWidth = ref(6) // 悬停时线条粗细
const chartWidth = ref('100%') // 图表宽度（支持像素、百分比、vw等）
const chartHeight = ref('650px') // 图表高度（支持像素、vh等）

/* Custom CSV order */
const preferredCsvOrder = ref([
  'm6A.csv',
  'ψ.csv',
  'm5C.csv',
  'AtoI.csv',
  'm7G.csv',
  'm1A.csv',
  // Add other CSV filenames in desired order
]);

/* Chart background color based on color scheme */
// const chartBackgroundColor = computed(() => {
//   return window.matchMedia('(prefers-color-scheme: dark)').matches ? '#1a1a1a' : '#ffffff';
// });

/* Chart background color (always white) */
const chartBackgroundColor = ref('#ffffff');

/* ---------- 2. 工具函数 -------- */
function generateColors(n) {
  return Array.from({ length: n }, (_, i) =>
    `hsl(${Math.round((360 / n) * i)}, 70%, 60%)`
  );
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
    const lines = csv.trim().split('\n');
    if (lines.length < 2) throw new Error('CSV file is empty or invalid');
    const _lbls = lines[0].split(',').slice(1).map(label => label.trim());
    if (!_lbls.length) throw new Error('No labels found in CSV');
    const modelData = {};
    for (let i = 1; i < lines.length; i++) {
      const cols = lines[i].split(',');
      if (cols.length !== _lbls.length + 1) throw new Error(`Invalid row format at line ${i + 1}`);
      const values = cols.slice(1).map(val => {
        const num = Number(val);
        // if (isNaN(num)) throw new Error(`Non-numeric value "${val}" in row ${i + 1}`);
        return num;
      });
      modelData[cols[0].trim()] = values;
    }
    return { labels: _lbls, data: modelData };
  } catch (error) {
    throw new Error(`CSV parsing error: ${error.message}`);
  }
}

/* ---------- 3. 标签到图片的映射 -------- */
const labelImageMap = computed(() => {
  // Map labels to image paths (adjust based on your CSV labels)
  const map = {};
  labels.value.forEach(label => {
    // Use lowercase and remove spaces for image filenames
    const imageName = label.toLowerCase().replace(/\s+/g, '') + '.png';
    // Images at /icons/<label>.png
    // map[label] = `/icons/${imageName}`; // public 好像不用写? 但是不加上又无法显示
    map[label] = `public/icons/${imageName}`;
  });
  return map;
});

/* ---------- 4. 图片预加载 -------- */
function preloadImages(imageMap) {
  Object.entries(imageMap).forEach(([label, src]) => {
    if (!imageCache.value[src]) {
      const img = new Image();
      img.src = src;
      img.onload = () => {
        imageCache.value[src] = img;
        if (chartRef.value?.chart) {
          chartRef.value.chart.update();
        }
      };
      img.onerror = () => {
        console.error(`Failed to load image: ${src}`);
        imageCache.value[src] = null; // 标记失败
      };
    }
  });
}

/* ---------- 5. 计算模型列 -------- */
const modelColumns = computed(() => {
  const models = modelNames.value;
  if (models.length <= 6) return [models]; // 少于等于6个模型，单列
  const columnCount = Math.ceil(models.length / 6); // 每列最多6个模型
  const columns = [];
  for (let i = 0; i < columnCount; i++) {
    columns.push(models.slice(i * 6, (i + 1) * 6));
  }
  return columns;
});

/* ---------- 6. 模型颜色映射 -------- */
const modelColors = computed(() => {
  const colorsMap = {};
  datasets.value.forEach(ds => {
    colorsMap[ds.label] = ds.borderColor;
  });
  return colorsMap;
});

/* ---------- 7. 加载 CSV 文件 -------- */
onMounted(async () => {
  try {
    // 使用 Vite 的 import.meta.glob 动态加载 CSV 文件
    console.log('Starting onMounted');
    const modules = import.meta.glob('/public/*.csv', { as: 'raw', eager: true });
    const availableFiles = Object.keys(modules).map(file => file.split('/').pop());
    console.log('Available CSV files:', availableFiles);

    // Sort files based on preferredCsvOrder, with unspecified files appended alphabetically
    const orderedFiles = [
      ...preferredCsvOrder.value.filter(file => availableFiles.includes(file)),
      ...availableFiles
        .filter(file => !preferredCsvOrder.value.includes(file))
        .sort((a, b) => a.localeCompare(b))
    ];
    csvFiles.value = orderedFiles;
    selectedCsv.value = csvFiles.value[0] || null; // 默认选中第一个文件
    console.log('Ordered CSV files:', csvFiles.value);

    if (!selectedCsv.value) {
      throw new Error('No CSV files found in public/');
    }

    // 缓存所有CSV文件数据
    for (const file of csvFiles.value) {
      csvData.value[file] = csvToJson(modules[`/public/${file}`]);
    }

    const initialCsv = selectedCsv.value;
    const { labels: _lbls, data: modelData } = csvData.value[initialCsv] || { labels: [], data: {} };
    const entries = Object.entries(modelData).filter(([name]) => !name.includes('Max') && !name.includes('Min'));
    const colors = generateColors(entries.length);

    labels.value = _lbls;
    datasets.value = entries.map(([name, values], i) => ({
      label: name,
      data: values,
      borderColor: colors[i],
      backgroundColor: colors[i].replace('hsl', 'hsla').replace(')', ', 0.2)'),
      pointBackgroundColor: colors[i],
      borderWidth: borderWidth.value,
      pointRadius: pointRadius.value,
      pointHoverRadius: pointHoverRadius.value,
      hoverBorderWidth: hoverBorderWidth.value,
      fill: true,
      pointHitRadius: 10,
      originalBorderColor: colors[i],
      originalBackgroundColor: colors[i].replace('hsl', 'hsla').replace(')', ', 0.2)')
    }));
    modelNames.value = entries.map(([name]) => name);
    // selectedModels.value = []
    // ready.value = true
    selectedModels.value = modelNames.value; // 默认全选 2025-07-12 17:40:16

    console.log('Data loaded for', initialCsv, { labels: _lbls, datasets: entries });

    preloadImages(labelImageMap.value);
    // console.log('labelImageMap.value: ', labelImageMap.value)
    console.log('Label image map:', labelImageMap.value);
    //  默认全选 2025-07-12 17:40:20
    ready.value = true;

    // 添加 mouseleave 事件监听器
    const canvas = chartRef.value?.chart?.canvas;
    if (canvas) {
      canvas.addEventListener('mouseleave', () => {
        console.log('Mouse left chart');
        hoveredDatasetIndex.value = null;
        hoveredDataIndex.value = null;
      });
    } else {
      console.error('Canvas not found');
    }
  } catch (error) {
    errorMessage.value = `Initialization failed: ${error.message}`; // 2025-07-12 17:41:58
    console.error('Error loading CSVs:', error);
  }
});

/* 监听 CSV(modification) 切换 */
watch(selectedCsv, (newCsv) => {
  try {
    errorMessage.value = null;
    ready.value = false;

    if (!csvData.value[newCsv]) {
      throw new Error(`Data for ${newCsv} not found in cache`);
    }

    const { labels: _lbls, data: modelData } = csvData.value[newCsv];
    const entries = Object.entries(modelData).filter(([name]) => !name.includes('Max') && !name.includes('Min'));
    const colors = generateColors(entries.length);

    labels.value = _lbls;
    datasets.value = entries.map(([name, values], i) => ({
      label: name,
      data: values,
      borderColor: colors[i],
      backgroundColor: colors[i].replace('hsl', 'hsla').replace(')', ', 0.2)'),
      pointBackgroundColor: colors[i],
      borderWidth: borderWidth.value,
      pointRadius: pointRadius.value,
      pointHoverRadius: pointHoverRadius.value,
      hoverBorderWidth: hoverBorderWidth.value,
      fill: true,
      pointHitRadius: 10,
      originalBorderColor: colors[i],
      originalBackgroundColor: colors[i].replace('hsl', 'hsla').replace(')', ', 0.2)')
    }));
    modelNames.value = entries.map(([name]) => name);
    selectedModels.value = modelNames.value;

    console.log('Data loaded for', newCsv, { labels: _lbls, datasets: entries });

    preloadImages(labelImageMap.value);
    console.log('Label image map:', labelImageMap.value);

    ready.value = true;
    if (chartRef.value?.chart) {
      chartRef.value.chart.update();
    }
  } catch (error) {
    errorMessage.value = `Failed to load ${newCsv}: ${error.message}`;
    console.error('Error loading CSV:', error);
    ready.value = true;
  }
});

/* 清理事件监听器 */
onUnmounted(() => {
  const canvas = chartRef.value?.chart?.canvas;
  if (canvas) {
    canvas.removeEventListener('mouseleave', () => {
      hoveredDatasetIndex.value = null;
      hoveredDataIndex.value = null;
    });
  }
});

/* ---------- 8. 计算图表数据（随CSV和模型选择变化） -------- */
const chartData = computed(() => {
  if (!selectedCsv.value || !csvData.value[selectedCsv.value]) {
    console.warn('No data for chartData');
    return { labels: [], datasets: [] };
  }

  // 获取当前选中的CSV文件数据
  const { labels: _lbls, data: modelData } = csvData.value[selectedCsv.value];
  const entries = Object.entries(modelData).filter(([name]) => !name.includes('Max') && !name.includes('Min'));
  const colors = generateColors(entries.length);

  // 更新 labels 和 datasets
  // 避免重复更新 labels.value
  if (JSON.stringify(labels.value) !== JSON.stringify(_lbls)) {
    labels.value = _lbls;
  }
  datasets.value = entries.map(([name, values], i) => ({
    label: name,
    data: values,
    borderColor: colors[i],
    backgroundColor: colors[i].replace('hsl', 'hsla').replace(')', ', 0.2)'),
    pointBackgroundColor: colors[i],
    borderWidth: borderWidth.value,
    pointRadius: pointRadius.value,
    pointHoverRadius: pointHoverRadius.value,
    hoverBorderWidth: hoverBorderWidth.value,
    fill: true,
    pointHitRadius: 10,
    originalBorderColor: colors[i],
    originalBackgroundColor: colors[i].replace('hsl', 'hsla').replace(')', ', 0.2)')
  }));
  modelNames.value = entries.map(([name]) => name);
  // 过滤选中的模型
  // 固定第一个标签，剩余标签逆时针反转
  const chartLabels = labels.value.length > 0
    ? [labels.value[0], ...labels.value.slice(1).reverse()]
    : [];
  console.log('chartLabels: ', chartLabels, 'original: ', [...labels.value]);
  const selectedDatasets = datasets.value
    .filter(ds => selectedModels.value.includes(ds.label))
    .map((ds, i) => {
      const isHovered = hoveredDatasetIndex.value === i;
      // 固定第一个数据点，剩余数据点反转
      const newData = ds.data.length > 0
        ? [ds.data[0], ...ds.data.slice(1).reverse()]
        : [];
      return {
        ...ds,
        data: newData,
        borderColor: isHovered ? ds.originalBorderColor : ds.originalBorderColor.replace('hsl', 'hsla').replace(')', ', 0.3)'),
        backgroundColor: isHovered ? ds.originalBackgroundColor : ds.originalBackgroundColor.replace('0.2)', '0)'),
        borderWidth: isHovered ? hoverBorderWidth.value : borderWidth.value,
        pointRadius: isHovered ? pointHoverRadius.value : pointRadius.value
      };
    });

  const chartDataObject = {
    labels: chartLabels,
    datasets: selectedDatasets
  };
  console.log('Chart data:', chartDataObject);
  return chartDataObject;
});

/* ---------- 9. 图表配置 -------- */
const chartOptions = computed(() => {
  const width = chartRef.value?.chart?.width || parseInt(chartWidth.value) || 700;
  const fontSize = Math.max(10, Math.min(14, width / 40));
  const padding = width / 6; // 增加内边距以容纳图片
  // const padding = 200; // 增加内边距以容纳图片
  const imageSize = Math.max(20, width / 15); // 调整图片大小
  const extraOffset = imageSize * 0.5 + 20; // 图片与标签额外间距

  return {
    responsive: true,
    maintainAspectRatio: false,
    // 动态调整间距, 防止标签被裁剪
    layout: { padding: padding },
    // 仅检测单个数据点
    interaction: { mode: 'point', intersect: true, includeInvisible: false },
    scales: {
      r: {
        // beginAtZero: true,
        min: -0.25, // 圆心为 -0.25，最内圈为 0
        max: 1, // 外圈为 1
        // grid: { circular: true },
        grid: { circular: false }, /* Changed to polylines */
        pointLabels: {
          font: { size: fontSize },
          padding: 0  // label 位置
        },
        ticks: {
          stepSize: 0.25, // 步长 0.25，生成刻度 [-0.25, 0, 0.25, 0.5, 0.75, 1]
          font: { size: fontSize - 2 },
          callback: value => value >= 0 ? value : null // 隐藏负值刻度
        }
      }
    },
    plugins: {
      legend: { display: false },
      tooltip: {
        enabled: true,
        position: 'nearest', // 使用默认定位器
        backgroundColor: 'rgba(0, 0, 0, 0.7)', // 半透明背景
        bodyFont: { size: fontSize - 2 },
        padding: fontSize / 2,
        callbacks: {
          title: () => '', // 移除标题
          label: ctx => {
            // 只显示当前悬停的数据点
            if (ctx.datasetIndex === hoveredDatasetIndex.value && ctx.dataIndex === hoveredDataIndex.value) {
              const value = Number(ctx.raw).toFixed(4) // 四舍五入到4位小数
              const label = `${ctx.chart.data.labels[ctx.dataIndex]}: ${value}`;
              console.log('Tooltip:', label);
              return label;
            }
            return '';
          }
        }
      },
      background: {
        id: 'background',
        beforeDraw: (chart) => {
          const { ctx, width, height } = chart;
          ctx.save();
          ctx.fillStyle = chartBackgroundColor.value;
          ctx.fillRect(0, 0, width, height);
          ctx.restore();
        }
      },
      lineHoverPlugin: {
        id: 'lineHoverPlugin',
        beforeEvent(chart, args) {
          const event = args.event;
          if (event.type === 'mousemove') {
            const { ctx, chartArea } = chart;
            const datasets = chart.data.datasets;
            let closestDatasetIndex = null;
            let minDistance = Infinity;

            datasets.forEach((dataset, datasetIndex) => {
              if (!chart.getDatasetMeta(datasetIndex).visible) return;
              const meta = chart.getDatasetMeta(datasetIndex);
              const elements = meta.data;
              const points = elements.map((el, i) => ({
                x: el.x,
                y: el.y,
                value: dataset.data[i]
              }));

              for (let i = 0; i < points.length - 1; i++) {
                const p1 = points[i];
                const p2 = points[i + 1] || points[0];
                const distance = pointToLineDistance(event.x, event.y, p1.x, p1.y, p2.x, p2.y);
                if (distance < minDistance && distance < 10) {
                  minDistance = distance;
                  closestDatasetIndex = datasetIndex;
                }
              }
            });

            hoveredDatasetIndex.value = closestDatasetIndex;
            hoveredDataIndex.value = null; // 无数据点索引，因为线条悬停不触发tooltip
            console.log('Line hover dataset:', closestDatasetIndex);
          }
        }
      },
      // Custom plugin to draw images next to point labels
      pointLabelImages: {
        id: 'pointLabelImages',
        afterDraw(chart) {
          const { ctx, scales: { r }, width } = chart;
          const labels = chart.data.labels;
          const labelPositions = r._pointLabelItems;
          const canvas = chart.canvas;
          if (!labelPositions || !canvas) return;

          const imageSize = Math.max(15, Math.min(width / 12, width / labels.length)); // 动态图片大小
          const centerX = r.xCenter;
          const centerY = r.yCenter;
          const fontSize = Math.max(10, Math.min(14, width / 40));

          ctx.font = `${fontSize}px sans-serif`;

          labels.forEach((label, index) => {
            const imageSrc = labelImageMap.value[label];
            if (!imageSrc || !imageCache.value[imageSrc]) return;

            const img = imageCache.value[imageSrc];
            if (img && img.complete) {
              const { x, y, textAlign } = labelPositions[index];
              const textWidth = ctx.measureText(label).width;
              const rad = Math.atan2(y - centerY, x - centerX);
              const deg = rad * 180 / Math.PI;
              let imageX, imageY;

              imageX = x + 1.0 * Math.cos(rad) * (textWidth + padding * 5) - imageSize / 2;
              imageY = y + 1.2 * Math.sin(rad) * (padding * 4) - imageSize / 2 + Math.abs(Math.cos(rad)) * 30;

              // 限制图片在画布边界内
              imageX = Math.max(0, Math.min(canvas.width - imageSize, imageX));
              imageY = Math.max(0, Math.min(canvas.height - imageSize, imageY));

              ctx.drawImage(img, imageX, imageY, imageSize, imageSize);
              console.log('image:', img, 'x:', x, 'y:', y, 'imageX:', imageX, 'imageY:', imageY, 'Size:', imageSize,
                'textAlign:', textAlign, 'radian:', rad, 'degree:', deg);
            }
          });
        }
      }
    },
    onHover: (event, chartElements) => {
      console.log('Hover elements:', chartElements);
      if (chartElements.length > 0) {
        const datasetIndex = chartElements[0].datasetIndex;
        const dataIndex = chartElements[0].index;
        hoveredDatasetIndex.value = datasetIndex;
        hoveredDataIndex.value = dataIndex;
      } else if (hoveredDataIndex.value !== null) {
        hoveredDatasetIndex.value = null;
        hoveredDataIndex.value = null;
      }
    }
  };
});

/* ---------- 10. 模型全选/取消全选逻辑 -------- */
const isAllModelsSelected = computed(() =>
  selectedModels.value.length === modelNames.value.length
);

function toggleAllModels() {
  selectedModels.value = isAllModelsSelected.value ? [] : [...modelNames.value];
}

/* ---------- 11. 注册自定义插件 -------- */
ChartJS.register({
  id: 'pointLabelImages',
  afterDraw: chartOptions.value.plugins.pointLabelImages.afterDraw
  // }, {
  //   id: 'lineHoverPlugin',
  //   beforeEvent: chartOptions.plugins.lineHoverPlugin.beforeEvent
  // //加上这段使得悬停无法高亮, 悬停并点击才能高亮
}, {
  id: 'background',
  beforeDraw: chartOptions.value.plugins.background.beforeDraw
});

/* ---------- 12. 计算点到线段距离 -------- */
function pointToLineDistance(px, py, x1, y1, x2, y2) {
  const dx = x2 - x1;
  const dy = y2 - y1;
  const lenSquared = dx * dx + dy * dy;
  if (lenSquared === 0) return Math.sqrt((px - x1) ** 2 + (py - y1) ** 2);

  const t = Math.max(0, Math.min(1, ((px - x1) * dx + (py - y1) * dy) / lenSquared));
  const projX = x1 + t * dx;
  const projY = y1 + t * dy;
  return Math.sqrt((px - projX) ** 2 + (py - projY) ** 2);
}
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  max-width: 1200px;
  margin: 0 auto;
  padding: 1rem;
  background-color: white; /* Light mode background */
}

.wrapper {
  width: 100%;
  max-width: v-bind(chartWidth);
  /* 使用 chartWidth 控制最大宽度 */
}

.chart-box {
  width: v-bind(chartWidth);
  /* 图表宽度 */
  height: v-bind(chartHeight);
  /* 图表高度 */
  position: relative;
}

.chart-box canvas {
  width: 100% !important;
  height: 100% !important;
}

.filters-container {
  display: flex;
  gap: 2rem;
  justify-content: flex-start;
  width: 100%;
  max-width: 1200px;
  /* 放宽宽度限制，允许模型区域扩展 */
  margin-top: 1rem;
}

.filters {
  text-align: start;
  width: auto;
  /* 移除固定宽度，允许自适应 */
}

.filters fieldset {
  padding: 0.5rem;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.filters legend {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  /* 文字与按钮间距 */
  font-size: 1rem;
}

.model-columns {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
  /* 允许列换行 */
}

.column {
  flex: 1;
  min-width: 120px;
  /* 每列最小宽度 */
  max-width: 200px;
  /* 限制单列最大宽度 */
}

.checkbox {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin: 0.3rem 0;
  cursor: pointer;
  white-space: nowrap; /* 防止模型名称换行 */
  padding: 0.2rem 0.5rem;
  border: 5px solid transparent;
  border-radius: 4px;
  position: relative;
}

.checkbox input[type="checkbox"] {
  position: absolute;
  opacity: 0;
  width: 0;
  height: 0;
  appearance: none; /* Remove browser-specific styling */
  -webkit-appearance: none;
  -moz-appearance: none;
}

.checkbox.selected {
  border-color: inherit; /* Uses the borderColor from :style */
  background-color: rgba(0, 0, 0, 0.05);
}

/* .checkbox.selected::before {
  content: '✓';
  display: inline-block;
  width: 12px;
  height: 12px;
  font-size: 10px;
  line-height: 12px;
  text-align: center;
  color: white;
  background-color: v-bind('modelColors[name] || "#ccc"');
  border-radius: 2px;
  margin-right: 0.3rem;
} */

.model-name {
  display: block;
  max-width: 150px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.toggle-btn {
  padding: 0.2rem 0.4rem;
  /* 缩小按钮尺寸以适应legend */
  font-size: 0.85rem;
  cursor: pointer;
  background-color: #f0f0f0;
  border: 1px solid #ccc;
  border-radius: 4px;
  line-height: 1.2;
}

.error {
  color: red;
  padding: 1rem;
  text-align: center;
}

/* Dark mode styles */
@media (prefers-color-scheme: dark) {
  .container {
    /* Dark mode background */
    /* background-color: #1a1a1a; */
    color: #e0e0e0; /* Light text for contrast */
  }

  .filters fieldset {
    border-color: #555; /* Lighter border for dark mode */
  }

  .filters legend {
    color: #e0e0e0; /* Light text for legend */
  }

  .checkbox {
    color: #e0e0e0; /* Light text for labels */
  }

  .checkbox.selected {
    background-color: rgba(255, 255, 255, 0.1); /* Slightly lighter background for selected */
  }

  .toggle-btn {
    background-color: #333; /* Darker button background */
    border-color: #555; /* Lighter border */
    color: #e0e0e0; /* Light text */
  }

  .error {
    color: #ff5555; /* Brighter red for visibility */
  }
}

/* 响应式设计：小屏幕 */
@media (max-width: 768px) {
  .container {
    padding: 0.5rem;
    /* Ensure white background on small screens */
    background-color: white; 
  }

  .wrapper {
    max-width: 100%;
  }

  .chart-box {
    width: 100%;
    /* 小屏幕下宽度自适应 */
    height: calc(v-bind(chartHeight) * 0.8);
    /* 高度缩小至 80% */
  }

  .filters-container {
    flex-direction: column;
    align-items: center;
    gap: 1rem;
  }

  .filters {
    width: 100%;
    max-width: 300px;
  }

  .filters legend {
    flex-wrap: wrap;
    /* 小屏幕允许换行 */
    gap: 0.3rem;
    font-size: 0.9rem;
  }

  .model-columns {
    flex-direction: column;
    /* 小屏幕垂直排列 */
  }

  .column {
    min-width: 100%;
    /* 小屏幕单列占满宽度 */
    max-width: 100%;
  }

  .checkbox {
    font-size: 0.8rem; /* 缩小字体 */
    padding: 0.15rem 0.4rem;
    border-width: 1px;
  }

  .checkbox.selected::before {
    width: 10px;
    height: 10px;
    font-size: 8px;
    line-height: 10px;
  }

  .model-name {
    max-width: 120px;
  }

  .toggle-btn {
    padding: 0.15rem 0.3rem;
    font-size: 0.75rem;
  }

  /* Dark mode styles for small screens */
  @media (prefers-color-scheme: dark) {
    .container {
      /* Dark mode background */
      /* background-color: #1a1a1a; */
      color: #e0e0e0;
    }

    .filters fieldset {
      border-color: #555;
    }

    .filters legend {
      color: #e0e0e0;
    }

    .checkbox {
      color: #e0e0e0;
    }

    .checkbox.selected {
      background-color: rgba(255, 255, 255, 0.1);
    }

    .toggle-btn {
      background-color: #333;
      border-color: #555;
      color: #e0e0e0;
    }

    .error {
      color: #ff5555;
    }
  }
}
</style>
