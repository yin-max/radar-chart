<template>
  <div v-if="errorMessage" class="error" role="alert">{{ errorMessage }}</div>
  <div v-else-if="!ready">Loading...</div>
  <div v-else class="container">
    <!-- ② Kit 选择区域、CSV 文件选择区域和模型选择区域 -->
    <div class="filters-container">
      <!-- Kit 选择区域 -->
      <div class="filters">
        <fieldset>
          <legend>Kit version</legend>
          <label v-for="kit in kitOptions" :key="kit" class="checkbox" :class="{ selected: selectedKit === kit }" :style="{ borderColor: selectedKit === kit ? '#000000' : 'transparent', color: selectedKit === kit ? '#000000' : '#585858' }">
            <input type="radio" :value="kit" v-model="selectedKit" name="kit-selection" :aria-label="`Select kit ${kit}`" />
            {{ kit }}
          </label>
        </fieldset>
      </div>

      <!-- CSV 文件选择区域 -->
      <div class="filters">
        <fieldset>
          <legend>RNA modifications</legend>
          <!-- 使用单选按钮选择单个CSV文件，显示时去掉 .csv 后缀 -->
          <label v-for="csv in csvFiles" :key="csv" class="checkbox" :class="{ selected: selectedCsv === csv }" :style="{ borderColor: selectedCsv === csv ? '#000000' : 'transparent', color: selectedCsv === csv ? '#000000' : '#585858' }">
            <input type="radio" :value="csv" v-model="selectedCsv" name="csv-selection" :aria-label="`Select CSV ${csv.replace('.csv', '')}`" />
            {{ csv.replace('.csv', '') }}
          </label>
        </fieldset>
      </div>

      <!-- 模型选择区域 -->
      <div class="filters">
        <fieldset>
          <!-- 🔘 一键切换按钮 -->
          <legend>
            Models
            <button @click="toggleAllModels" class="toggle-btn" :aria-label="isOneModelSelected ? 'Clear all models' : 'Select all models'">
              {{ isOneModelSelected ? 'Clear' : 'All' }}
            </button>
          </legend>
          <!-- ✅ 动态复选框 -->
          <div class="model-columns">
            <div v-for="(column, colIndex) in modelColumns" :key="colIndex" class="column">
              <!-- <label v-for="name in column" :key="name" class="checkbox" :class="{ selected: selectedModels.includes(name) }" :style="{ backgroundColor: selectedModels.includes(name) ? modelColors[name] || '#ccc' : 'transparent' }" :title="name"> -->
              <label v-for="name in column" :key="name" class="checkbox" :class="{ selected: selectedModels.includes(name) }" :style="{ backgroundColor: selectedModels.includes(name) ? modelColors[name] || '#ccc' : 'transparent', textShadow: selectedModels.includes(name) ? '1px 1px 2px rgba(0, 0, 0, 0.5)' : 'none' }" :title="name">
              <!-- <label v-for="name in column" :key="name" class="checkbox"> -->
                <!-- <span class="color-indicator" :style="{ backgroundColor: modelColors[name] || '#ccc' }"></span> -->
                <input type="checkbox" :value="name" v-model="selectedModels" :aria-label="`Select model ${name}`" />
                <span class="model-name" :style="{ color: selectedModels.includes(name) ? '#ffffff' : modelColors[name] || '#ccc' }">{{ name }}</span>
              </label>
            </div>
          </div>
        </fieldset>
      </div>
    </div>

    <!-- ① 雷达图 -->
    <div class="wrapper">
      <div class="chart-box">
        <Radar :data="chartData" :options="chartOptions" ref="chartRef" />
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
const ready = ref(false);
const errorMessage = ref(null);
const kitOptions = ref([]); // Kit 文件夹列表（SQK-RNA002, SQK-RNA004 等）
const selectedKit = ref(null); // 当前选中的 Kit
const csvFiles = ref([]); // 当前 Kit 下的 CSV 文件列表
const selectedCsv = ref(null); // 当前选中的 CSV 文件
const csvData = ref({}); // 缓存所有 CSV 文件的数据
const labels = ref([]); // 当前选中的 CSV 文件的指标
const datasets = ref([]); // 当前选中的 CSV 文件的模型数据集
const modelNames = ref([]); // 当前选中的 CSV 文件的模型名称数组
const selectedModels = ref([]); // 当前选中的模型
const hoveredDatasetIndex = ref(null); // 当前高亮的 dataset 索引
const hoveredDataIndex = ref(null); // 当前高亮的数据点索引
const chartRef = ref(null); // 引用 Radar 组件
const imageCache = ref({}); // 图片缓存
const maxModelNumPerColumn = ref(6); // 每个列最多显示的模型数量
const pointRadius = ref(2); // 默认点大小（像素）
const pointHoverRadius = ref(4); // 悬停时点大小
const borderWidth = ref(4); // 默认线条粗细
const hoverBorderWidth = ref(6); // 悬停时线条粗细
const chartWidth = ref('100%'); // 图表宽度
const chartHeight = ref('650px'); // 图表高度
const chartBackgroundColor = ref('#ffffff'); // 图表背景色

/* HEX 到 HSL 转换函数 */
function hexToHsl(hex) {
  hex = hex.replace('#', '');
  const r = parseInt(hex.slice(0, 2), 16) / 255;
  const g = parseInt(hex.slice(2, 4), 16) / 255;
  const b = parseInt(hex.slice(4, 6), 16) / 255;

  const max = Math.max(r, g, b);
  const min = Math.min(r, g, b);
  let h, s, l = (max + min) / 2;

  if (max === min) {
    h = s = 0;
  } else {
    const d = max - min;
    s = l > 0.5 ? d / (2 - max - min) : d / (max + min);
    switch (max) {
      case r: h = (g - b) / d + (g < b ? 6 : 0); break;
      case g: h = (b - r) / d + 2; break;
      case b: h = (r - g) / d + 4; break;
    }
    h /= 6;
  }

  h = Math.round(h * 360);
  s = Math.round(s * 100);
  l = Math.round(l * 100);
  return `hsl(${h}, ${s}%, ${l}%)`;
}

/* 模型颜色映射（HSL 格式） */
const modelColorMap = {
  'm6Anet': hexToHsl('#2e3792'), // hsl(236, 54%, 38%)
  'm6Anet-retrain': hexToHsl('#2e3792'), // hsl(236, 54%, 38%)
  'EpiNano': hexToHsl('#8e5aa2'), // hsl(286, 32%, 49%)
  'EpiNano-retrain': hexToHsl('#8e5aa2'), // hsl(286, 32%, 49%)
  'SingleMod': hexToHsl('#f6c365'), // hsl(40, 89%, 67%)
  'SingleMod-retrain': hexToHsl('#f6c365'), // hsl(40, 89%, 67%)
  'NanoSPA': hexToHsl('#bc1932'), // hsl(351, 78%, 43%)
  'NanoSPA-retrain': hexToHsl('#bc1932'), // hsl(351, 78%, 43%)
  'TandemMod': hexToHsl('#9DCB62'), // hsl(90, 50%, 60%)
  'TandemMod-retrain': hexToHsl('#9DCB62'), // hsl(90, 50%, 60%)
  'Dinopore': hexToHsl('#F4B5CA'), // hsl(342, 77%, 83%)
  'Dinopore-retrain': hexToHsl('#F4B5CA'), // hsl(342, 77%, 83%)
  'Nanom6A': hexToHsl('#098889'), // hsl(181, 88%, 30%)
  'ELIGOS': hexToHsl('#cca814'), // hsl(46, 83%, 44%)
  'ELIGOS_diff': hexToHsl('#c5781a'), // hsl(33, 78%, 45%)
  'MINES': hexToHsl('#7b5223'), // hsl(31, 55%, 31%)
  'Epinano_delta': hexToHsl('#c896c8'), // hsl(300, 31%, 68%)
  'CHEUI': hexToHsl('#6ab93c'), // hsl(101, 50%, 47%)
  'Tombo': hexToHsl('#57217b'), // hsl(279, 57%, 30%)
  'Tombo_com': hexToHsl('#b82373'), // hsl(325, 68%, 43%)
  'DiffErr': hexToHsl('#cfe298'), // hsl(79, 57%, 72%)
  'DRUMMER': hexToHsl('#d25a9c'), // hsl(326, 56%, 59%)
  'xPore': hexToHsl('#5d96d0'), // hsl(211, 53%, 60%)
  'Nanocompore': hexToHsl('#969696'), // hsl(0, 0%, 59%)
  'DENA': hexToHsl('#a8d6b3'), // hsl(137, 35%, 75%)
  'm6Aiso': hexToHsl('#f1881a'), // hsl(33, 89%, 52%)
  'pum6A': hexToHsl('#129abf'), // hsl(193, 81%, 43%)
  'NanoMUD': hexToHsl('#78862f'), // hsl(71, 50%, 34%)
  'NanoRMS': hexToHsl('#4b4b4b'), // hsl(0, 0%, 29%)
  'PsiNanopore': hexToHsl('#2a65b0'), // hsl(212, 60%, 43%)
  'Xron': hexToHsl('#6affb9'), // hsl(155, 100%, 71%)
  'Dorado': hexToHsl('#ef1fff'), // hsl(294, 100%, 56%)
};

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

/* ---------- 2. 工具函数 -------- */
function generateColors(n, modelNames) {
  const colors = [];
  for (let i = 0; i < n; i++) {
    const model = modelNames[i];
    if (modelColorMap[model]) {
      colors.push(modelColorMap[model]);
    } else {
      colors.push(`hsl(${Math.round((360 / n) * i)}, 70%, 60%)`);
    }
  }
  return colors;
}

function csvToJson(csv) {
  try {
    const lines = csv.trim().split('\n');
    if (lines.length < 2) throw new Error('CSV file is empty or invalid');
    const _lbls = lines[0].split(',').slice(1).map(label => label.trim());
    if (!_lbls.length) throw new Error('No labels found in CSV');
    const modelData = {};
    const sortData = {}; // 用于排序的数据，NA 转为 0
    for (let i = 1; i < lines.length; i++) {
      const cols = lines[i].split(',');
      if (cols.length !== _lbls.length + 1) throw new Error(`Invalid row format at line ${i + 1}`);
      const values = cols.slice(1).map(val => {
        const num = Number(val);
        // if (isNaN(num)) throw new Error(`Non-numeric value "${val}" in row ${i + 1}`);
        return num;
      });
      const sortValues = cols.slice(1).map(val => val.trim() === 'NA' ? 0 : Number(val));
      modelData[cols[0].trim()] = values;
      sortData[cols[0].trim()] = sortValues;
    }
    return { labels: _lbls, data: modelData, sortData };
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
    map[label] = `./icons/${imageName}`; // 生产环境中解析为 /icons/*.png
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
  if (models.length <= maxModelNumPerColumn.value) return [models]; // 少于等于6个模型，单列
  const columnCount = Math.ceil(models.length / maxModelNumPerColumn.value); // 每列最多6个模型
  const columns = [];
  for (let i = 0; i < columnCount; i++) {
    columns.push(models.slice(i * maxModelNumPerColumn.value, (i + 1) * maxModelNumPerColumn.value));
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
    // 动态加载 /public/SQK-RNA*/*.csv 文件
    const publicPath = import.meta.env.DEV ? '/public' : '';
    const modules = import.meta.glob('/public/SQK-RNA*/*.csv', { as: 'raw', eager: true });
    const availableFiles = Object.keys(modules).map(file => file.replace('/public/', ''));

    // 提取 Kit 文件夹（SQK-RNA*）
    const kits = [...new Set(availableFiles.map(file => file.split('/')[0]))]
      .filter(kit => kit.startsWith('SQK-RNA'))
      .sort();
    kitOptions.value = kits;
    selectedKit.value = kits[0] || null;
    console.log('Available kits:', kits);

    if (!selectedKit.value) {
      throw new Error('No SQK-RNA* folders found in public/');
    }

    // 初始化 CSV 文件列表
    updateCsvFiles();

    // 缓存所有 CSV 文件数据
    for (const file of availableFiles) {
      csvData.value[file] = csvToJson(modules[`${publicPath}/${file}`]);
    }

    // 加载初始 CSV 数据
    loadCsvData(selectedCsv.value);

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

/* 更新 CSV 文件列表（基于选中的 Kit） */
function updateCsvFiles() {
  const publicPath = import.meta.env.DEV ? '/public' : '';
  const modules = import.meta.glob('/public/SQK-RNA*/*.csv', { as: 'raw', eager: true });
  const availableFiles = Object.keys(modules).map(file => file.replace('/public/', ''));
  const kitFiles = availableFiles
    .filter(file => file.startsWith(`${selectedKit.value}/`))
    .map(file => file.split('/')[1])
    .sort((a, b) => {
      const indexA = preferredCsvOrder.value.indexOf(a);
      const indexB = preferredCsvOrder.value.indexOf(b);
      if (indexA === -1 && indexB === -1) return a.localeCompare(b);
      if (indexA === -1) return 1;
      if (indexB === -1) return -1;
      return indexA - indexB;
    });
  csvFiles.value = kitFiles;
  selectedCsv.value = kitFiles[0] || null;
  console.log('CSV files for', selectedKit.value, ':', kitFiles);
}

/* 加载 CSV 数据并按总和排序 */
function loadCsvData(csvFile) {
  if (!csvFile) return;
  const filePath = `${selectedKit.value}/${csvFile}`;
  if (!csvData.value[filePath]) {
    throw new Error(`Data for ${filePath} not found in cache`);
    }
  const { labels: _lbls, data: modelData, sortData } = csvData.value[filePath];

  // 计算每个模型的指标总和（NA 视为 0）
  const modelSums = Object.entries(sortData)
    .filter(([name]) => !name.includes('Max') && !name.includes('Min'))
    .map(([name, values]) => ({
      name,
      sum: values.reduce((acc, val) => acc + val, 0)
    }))
    .sort((a, b) => b.sum - a.sum); // 降序排序

  const sortedModelNames = modelSums.map(({ name }) => name);
  const colors = generateColors(sortedModelNames.length, sortedModelNames);

  labels.value = _lbls;
  datasets.value = sortedModelNames.map((name, i) => ({
    label: name,
    data: modelData[name], // 使用原始数据（NA 保持为 null）
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
  modelNames.value = sortedModelNames;
  selectedModels.value = sortedModelNames;
  console.log('Data loaded for', filePath, { labels: _lbls, datasets: sortedModelNames, sums: modelSums });
}

/* 监听 Kit 切换 */
watch(selectedKit, () => {
  try {
    errorMessage.value = null;
    ready.value = false;
    updateCsvFiles();
    loadCsvData(selectedCsv.value);
    preloadImages(labelImageMap.value);
    ready.value = true;
    if (chartRef.value?.chart) {
      chartRef.value.chart.update();
    }
  } catch (error) {
    errorMessage.value = `Failed to load kit ${selectedKit.value}: ${error.message}`;
    console.error('Error loading kit:', error);
    ready.value = true;
  }
});

/* 监听 CSV 切换 */
watch(selectedCsv, (newCsv) => {
  try {
    errorMessage.value = null;
    ready.value = false;
    loadCsvData(newCsv);
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
  if (!selectedCsv.value || !selectedKit.value || !csvData.value[`${selectedKit.value}/${selectedCsv.value}`]) {
    console.warn('No data for chartData');
    return { labels: [], datasets: [] };
  }

  // 获取当前选中的CSV文件数据
  const { labels: _lbls, data: modelData } = csvData.value[`${selectedKit.value}/${selectedCsv.value}`];
  // const { labels: _lbls, data: modelData } = csvData.value[selectedCsv.value];
  const entries = Object.entries(modelData).filter(([name]) => !name.includes('Max') && !name.includes('Min'));
  // 保持 modelNames 的排序顺序
  const colors = generateColors(modelNames.value.length, modelNames.value);

  // 更新 labels 和 datasets, 且避免重复更新 labels.value
  if (JSON.stringify(labels.value) !== JSON.stringify(_lbls)) {
    labels.value = _lbls;
  }
  datasets.value = modelNames.value.map((name, i) => ({
    label: name,
    data: modelData[name],
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
  // modelNames.value = entries.map(([name]) => name);
  // 过滤选中的模型
  // 固定第一个标签，剩余标签逆时针反转
  const chartLabels = labels.value.length > 0
    ? [labels.value[0], ...labels.value.slice(1).reverse()]
    : [];
  console.log('chartLabels:', chartLabels, 'original:', [...labels.value]);
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

const isOneModelSelected = computed(() =>
  selectedModels.value.length > 0
);

function toggleAllModels() {
  // selectedModels.value = isAllModelsSelected.value ? [] : [...modelNames.value];
  selectedModels.value = isOneModelSelected.value ? [] : [...modelNames.value];
}

/* ---------- 11. 注册自定义插件 -------- */
ChartJS.register({
  id: 'pointLabelImages',
  afterDraw: chartOptions.value.plugins.pointLabelImages.afterDraw
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
  height: v-bind(chartHeight);
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
  --model-color: v-bind('modelColors[name] || "#585858"');
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
  color: var(--model-color);
}

.checkbox input[type="checkbox"], .checkbox input[type="radio"] {
  position: absolute;
  opacity: 0;
  width: 0;
  height: 0;
  appearance: none; /* Remove browser-specific styling */
  -webkit-appearance: none;
  -moz-appearance: none;
}

.checkbox.selected {
  border-color: #000000;
  color: #ffffff;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
  background-color: inherit;
}

.checkbox .model-name {
  /* color: #ffffff; */
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
}


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
    /* color: #e0e0e0; Light text for contrast */
    color: #000000; /* Light text for contrast */
  }

  .filters fieldset {
    border-color: #555; /* Lighter border for dark mode */
  }

  .filters legend {
    /* color: #e0e0e0; Light text for legend */
    color: #000000; /* Light text for contrast */
  }

  .checkbox {
    --model-color: v-bind('modelColors[name] || "#585858"');
    color: var(--model-color);
  }

  .checkbox.selected {
    color: #ffffff;
    text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
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
      --model-color: v-bind('modelColors[name] || "#ccc"');
      color: var(--model-color);
    }

    .checkbox.selected {
      color: #ffffff;
      text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
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
