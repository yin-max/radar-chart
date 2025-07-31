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

    <!-- ② Kit 选择区域、CSV 文件选择区域和模型选择区域 -->
    <div class="filters-container">
      <!-- Kit 选择区域 -->
      <div class="filters">
        <fieldset>
          <legend>Kit version</legend>
          <label v-for="kit in kitOptions" :key="kit" class="checkbox" :class="{ selected: selectedKit === kit }" :style="{ borderColor: selectedKit === kit ? '#000000' : 'transparent', color: selectedKit === kit ? '#000000' : '#ccc' }">
            <input type="radio" :value="kit" v-model="selectedKit" name="kit-selection" :aria-label="`Select kit ${kit}`" />
            {{ kit }}
          </label>
        </fieldset>
      </div>

      <!-- CSV 文件选择区域 -->
      <div class="filters">
        <fieldset>
          <legend>RNA modifications</legend>
          <label v-for="csv in csvFiles" :key="csv" class="checkbox" :class="{ selected: selectedCsv === csv }" :style="{ borderColor: selectedCsv === csv ? '#000000' : 'transparent', color: selectedCsv === csv ? '#000000' : '#ccc' }">
            <input type="radio" :value="csv" v-model="selectedCsv" name="csv-selection" :aria-label="`Select CSV ${csv.replace('.csv', '')}`" />
            {{ csv.replace('.csv', '') }}
          </label>
        </fieldset>
      </div>

      <!-- 模型选择区域 -->
      <div class="filters">
        <fieldset>
          <legend>
            Models
            <button @click="toggleAllModels" class="toggle-btn" :aria-label="isOneModelSelected ? 'Clear all models' : 'Select all models'">
              {{ isOneModelSelected ? 'Clear' : 'All' }}
            </button>
          </legend>
          <div class="model-columns">
            <div v-for="(column, colIndex) in modelColumns" :key="colIndex" class="column">
              <label v-for="name in column" :key="name" class="checkbox" :class="{ selected: selectedModels.includes(name) }" :style="{ backgroundColor: selectedModels.includes(name) ? modelColors[name] || '#ccc' : 'transparent', textShadow: selectedModels.includes(name) ? '1px 1px 2px rgba(0, 0, 0, 0.5)' : 'none' }" :title="name">
                <input type="checkbox" :value="name" v-model="selectedModels" :aria-label="`Select model ${name}`" />
                <span class="model-name" :style="{ color: selectedModels.includes(name) ? '#ffffff' : modelColors[name] || '#ccc' }">{{ name }}</span>
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
const ready = ref(false);
const errorMessage = ref(null);
const kitOptions = ref([]);
const selectedKit = ref(null);
const csvFiles = ref([]);
const selectedCsv = ref(null);
const csvData = ref({});
const labels = ref([]);
const datasets = ref([]);
const modelNames = ref([]);
const selectedModels = ref([]);
const hoveredDatasetIndex = ref(null);
const hoveredDataIndex = ref(null);
const chartRef = ref(null);
const imageCache = ref({});
const maxModelNumPerColumn = ref(6);
const pointRadius = ref(2);
const pointHoverRadius = ref(4);
const borderWidth = ref(4);
const hoverBorderWidth = ref(6);
const chartWidth = ref('100%');
const chartHeight = ref('650px');
const chartBackgroundColor = ref('#ffffff');

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
  'EpiNano': hexToHsl('#8e5aa2'), // hsl(286, 32%, 49%)
  'SingleMod': hexToHsl('#f6c365'), // hsl(40, 89%, 67%)
  'NanoSPA (NanpPsu)': hexToHsl('#bc1932'), // hsl(351, 78%, 43%)
  'TandomMod': hexToHsl('#9fcc62'), // hsl(90, 50%, 60%)
  'DInoPORE': hexToHsl('#f5b6cb'), // hsl(342, 77%, 83%)
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
]);

/* ---------- 2. 工具函数 -------- */
function csvToJson(csv) {
  try {
    const lines = csv.trim().split('\n');
    if (lines.length < 2) throw new Error('CSV file is empty or invalid');
    const _lbls = lines[0].split(',').slice(1).map(label => label.trim());
    if (!_lbls.length) throw new Error('No labels found in CSV');
    const modelData = {};
    const sortData = {};
    for (let i = 1; i < lines.length; i++) {
      const cols = lines[i].split(',');
      if (cols.length !== _lbls.length + 1) throw new Error(`Invalid row format at line ${i + 1}`);
      const values = cols.slice(1).map(val => val.trim() === 'NA' ? null : Number(val));
      const sortValues = cols.slice(1).map(val => val.trim() === 'NA' ? 0 : Number(val));
      modelData[cols[0].trim()] = values;
      sortData[cols[0].trim()] = sortValues;
    }
    return { labels: _lbls, data: modelData, sortData };
  } catch (error) {
    throw new Error(`CSV parsing error: ${error.message}`);
  }
}

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

/* ---------- 3. 标签到图片的映射 -------- */
const labelImageMap = computed(() => {
  const map = {};
  const imagePath = '/icons';
  labels.value.forEach(label => {
    const imageName = label.toLowerCase().replace(/\s+/g, '') + '.png';
    map[label] = `${imagePath}/${imageName}`;
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
        imageCache.value[src] = null;
      };
    }
  });
}

/* ---------- 5. 计算模型列 -------- */
const modelColumns = computed(() => {
  const models = modelNames.value;
  if (models.length <= maxModelNumPerColumn.value) return [models];
  const columnCount = Math.ceil(models.length / maxModelNumPerColumn.value);
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
    console.log('Starting onMounted');
    // 预定义的 Kit 和 CSV 文件列表（可通过服务器 API 动态获取）
    const kits = ['SQK-RNA002', 'SQK-RNA004'];
    kitOptions.value = kits;
    selectedKit.value = kits[0] || null;
    console.log('Available kits:', kits);

    if (!selectedKit.value) {
      throw new Error('No SQK-RNA* folders found');
    }

    await updateCsvFiles();
    await loadCsvData(selectedCsv.value);
    preloadImages(labelImageMap.value);
    console.log('Label image map:', labelImageMap.value);
    ready.value = true;

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
    errorMessage.value = `Initialization failed: ${error.message}`;
    console.error('Error loading CSVs:', error);
  }
});

/* 更新 CSV 文件列表 */
async function updateCsvFiles() {
  try {
    // 假设 CSV 文件列表已知，也可通过 API 获取
    const kitFiles = preferredCsvOrder.value.filter(csv =>
      ['m6A.csv', 'ψ.csv'].includes(csv) // 根据实际文件调整
    );
    csvFiles.value = kitFiles;
    selectedCsv.value = kitFiles[0] || null;
    console.log('CSV files for', selectedKit.value, ':', kitFiles);

    // 预加载所有 CSV 文件
    for (const csv of kitFiles) {
      const filePath = `${selectedKit.value}/${csv}`;
      if (!csvData.value[filePath]) {
        const response = await fetch(`/data/${filePath}`);
        if (!response.ok) {
          console.warn(`Failed to fetch ${filePath}: ${response.statusText}`);
          continue;
        }
        const text = await response.text();
        csvData.value[filePath] = csvToJson(text);
      }
    }
  } catch (error) {
    console.error('Error updating CSV files:', error);
    csvFiles.value = [];
    selectedCsv.value = null;
  }
}

/* 加载 CSV 数据并按总和排序 */
async function loadCsvData(csvFile) {
  if (!csvFile) return;
  const filePath = `${selectedKit.value}/${csvFile}`;
  try {
    if (!csvData.value[filePath]) {
      const response = await fetch(`/data/${filePath}`);
      if (!response.ok) throw new Error(`Failed to fetch ${filePath}`);
      const text = await response.text();
      csvData.value[filePath] = csvToJson(text);
    }
    const { labels: _lbls, data: modelData, sortData } = csvData.value[filePath];

    // 计算每个模型的指标总和（NA 视为 0）
    const modelSums = Object.entries(sortData)
      .filter(([name]) => !name.includes('Max') && !name.includes('Min'))
      .map(([name, values]) => ({
        name,
        sum: values.reduce((acc, val) => acc + val, 0)
      }))
      .sort((a, b) => b.sum - a.sum);

    const sortedModelNames = modelSums.map(({ name }) => name);
    const colors = generateColors(sortedModelNames.length, sortedModelNames);

    labels.value = _lbls;
    datasets.value = sortedModelNames.map((name, i) => ({
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
    modelNames.value = sortedModelNames;
    selectedModels.value = sortedModelNames;
    console.log('Data loaded for', filePath, { labels: _lbls, datasets: sortedModelNames, sums: modelSums });
  } catch (error) {
    throw new Error(`Failed to load ${filePath}: ${error.message}`);
  }
}

/* 监听 Kit 切换 */
watch(selectedKit, async () => {
  try {
    errorMessage.value = null;
    ready.value = false;
    await updateCsvFiles();
    await loadCsvData(selectedCsv.value);
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
watch(selectedCsv, async (newCsv) => {
  try {
    errorMessage.value = null;
    ready.value = false;
    await loadCsvData(newCsv);
    preloadImages(labelImageMap.value);
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

/* ---------- 8. 计算图表数据 -------- */
const chartData = computed(() => {
  if (!selectedCsv.value || !selectedKit.value || !csvData.value[`${selectedKit.value}/${selectedCsv.value}`]) {
    console.warn('No data for chartData');
    return { labels: [], datasets: [] };
  }

  const { labels: _lbls, data: modelData } = csvData.value[`${selectedKit.value}/${selectedCsv.value}`];
  const entries = Object.entries(modelData).filter(([name]) => !name.includes('Max') && !name.includes('Min'));
  const colors = generateColors(modelNames.value.length, modelNames.value);

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

  const chartLabels = labels.value.length > 0
    ? [labels.value[0], ...labels.value.slice(1).reverse()]
    : [];
  console.log('chartLabels:', chartLabels, 'original:', [...labels.value]);
  const selectedDatasets = datasets.value
    .filter(ds => selectedModels.value.includes(ds.label))
    .map((ds, i) => {
      const isHovered = hoveredDatasetIndex.value === i;
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
  const padding = width / 6;
  const imageSize = Math.max(20, width / 15);
  const extraOffset = imageSize * 0.5 + 20;

  return {
    responsive: true,
    maintainAspectRatio: false,
    layout: { padding: padding },
    interaction: { mode: 'point', intersect: true, includeInvisible: false },
    scales: {
      r: {
        min: -0.25,
        max: 1,
        grid: { circular: false },
        pointLabels: {
          font: { size: fontSize },
          padding: 0
        },
        ticks: {
          stepSize: 0.25,
          font: { size: fontSize - 2 },
          callback: value => value >= 0 ? value : null
        }
      }
    },
    plugins: {
      legend: { display: false },
      tooltip: {
        enabled: true,
        position: 'nearest',
        backgroundColor: 'rgba(0, 0, 0, 0.7)',
        bodyFont: { size: fontSize - 2 },
        padding: fontSize / 2,
        callbacks: {
          title: () => '',
          label: ctx => {
            if (ctx.datasetIndex === hoveredDatasetIndex.value && ctx.dataIndex === hoveredDataIndex.value) {
              const value = Number(ctx.raw)?.toFixed(4) || 'NA';
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
            hoveredDataIndex.value = null;
            console.log('Line hover dataset:', closestDatasetIndex);
          }
        }
      },
      pointLabelImages: {
        id: 'pointLabelImages',
        afterDraw(chart) {
          const { ctx, scales: { r }, width } = chart;
          const labels = chart.data.labels;
          const labelPositions = r._pointLabelItems;
          const canvas = chart.canvas;
          if (!labelPositions || !canvas) return;

          const imageSize = Math.max(15, Math.min(width / 12, width / labels.length));
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
              let imageX = x + 1.0 * Math.cos(rad) * (textWidth + padding * 5) - imageSize / 2;
              let imageY = y + 1.2 * Math.sin(rad) * (padding * 4) - imageSize / 2 + Math.abs(Math.cos(rad)) * 30;

              imageX = Math.max(0, Math.min(canvas.width - imageSize, imageX));
              imageY = Math.max(0, Math.min(canvas.height - imageSize, imageY));

              ctx.drawImage(img, imageX, imageY, imageSize, imageSize);
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
  background-color: white;
}

.wrapper {
  width: 100%;
  max-width: v-bind(chartWidth);
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
  margin-top: 1rem;
}

.filters {
  text-align: start;
  width: auto;
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
  font-size: 1rem;
}

.model-columns {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
}

.column {
  flex: 1;
  min-width: 120px;
  max-width: 200px;
}

.checkbox {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin: 0.3rem 0;
  cursor: pointer;
  white-space: nowrap;
  padding: 0.2rem 0.5rem;
  border: 5px solid transparent;
  border-radius: 4px;
  position: relative;
}

.checkbox input[type="checkbox"], .checkbox input[type="radio"] {
  position: absolute;
  opacity: 0;
  width: 0;
  height: 0;
  appearance: none;
  -webkit-appearance: none;
  -moz-appearance: none;
}

.checkbox.selected {
  border-color: #000000;
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

@media (prefers-color-scheme: dark) {
  .container {
    color: #000000;
  }

  .filters fieldset {
    border-color: #555;
  }

  .filters legend {
    color: #000000;
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

@media (max-width: 768px) {
  .container {
    padding: 0.5rem;
    background-color: white;
  }

  .wrapper {
    max-width: 100%;
  }

  .chart-box {
    width: 100%;
    height: calc(v-bind(chartHeight) * 0.8);
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
    gap: 0.3rem;
    font-size: 0.9rem;
  }

  .model-columns {
    flex-direction: column;
  }

  .column {
    min-width: 100%;
    max-width: 100%;
  }

  .checkbox {
    font-size: 0.8rem;
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

  @media (prefers-color-scheme: dark) {
    .container {
      color: #e0e0e0;
    }

    .filters fieldset {
      border-color: #555;
    }

    .filters legend {
      color: #e0e0e0;
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