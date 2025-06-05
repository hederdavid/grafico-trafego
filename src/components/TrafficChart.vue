<template>
  <div class="chart-container">
    <div class="header">
      <h2 class="title">📊 Monitoramento de Tráfego de Rede</h2>
    </div>
    <div class="controls">
      <div class="stats">
        <div class="stat">
          <span class="stat-label">Taxa atual:</span>
          <span class="stat-value">{{ currentRate }} bytes/s</span>
        </div>
      </div>
      <div class="update-interval">
        <label for="interval-input"
          >Defina o intervalo de atualização (segundos):</label
        >
        <input
          id="interval-input"
          type="number"
          v-model="updateInterval"
          min="1"
        />
      </div>
      <div class="update-interval">
        <label for="interval-input">Defina o tamanho máximo do gráfico:</label>
        <input
          id="interval-input"
          type="number"
          v-model="historySize"
          min="1"
        />
      </div>
      <div class="chart-selection">
        <label for="chart-select">Selecione o gráfico:</label>
        <select id="chart-select" v-model="selectedChart">
          <option value="received">Bytes Recebidos</option>
          <option value="rate">Taxa de Transmissão</option>
        </select>
      </div>
    </div>
    <div class="chart-wrapper">
      <component
        :is="selectedChart === 'received' ? 'line-chart' : 'rate-chart'"
        :data="selectedChart === 'received' ? chartData : rateChartData"
        :options="chartOptions"
      />
    </div>
    <div class="footer">
      <span class="update-info">Atualizado em: {{ lastUpdate }}</span>
      <span class="connection-status" :class="{ connected: isConnected }">
        {{ isConnected ? "Conectado" : "Desconectado" }}
      </span>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, toRaw, watch } from "vue";
import axios from "axios";
import { Line } from "vue-chartjs";
import {
  Chart,
  LineElement,
  PointElement,
  LinearScale,
  Title,
  CategoryScale,
  Tooltip,
  Filler,
  Legend,
} from "chart.js";

const lastUpdate = ref("N/A");
const isConnected = ref(true);
const previousData = ref(0);
const updateInterval = ref(5); // Valor inicial em segundos
const selectedChart = ref("received"); // Gráfico inicial
let updateIntervalId = null;
const historySize = ref(20); // Tamanho máximo do histórico de dados

const currentRate = computed(() => {
  const data = chartData.value.datasets[0].data;
  const labels = chartData.value.labels;

  if (data.length < 2 || labels.length < 2) return "0";

  const lastValue = data[data.length - 1];
  const previousValue = data[data.length - 2];

  const rate = (lastValue - previousValue) / updateInterval.value;
  return rate.toLocaleString();
});

const suggestedMax = computed(() => {
  const data = chartData.value.datasets[0].data;
  const maxValue = Math.max(...data, 0);
  return Math.ceil(maxValue * 1.2);
});

Chart.register(
  LineElement,
  PointElement,
  LinearScale,
  Title,
  CategoryScale,
  Tooltip,
  Filler,
  Legend
);

const chartData = ref({
  labels: [],
  datasets: [
    {
      label: "Bytes recebidos",
      data: [],
      borderColor: "#1e90ff",
      backgroundColor: null,
      fill: true,
      tension: 0.4,
      pointRadius: 5,
      pointHoverRadius: 8,
      pointBorderWidth: 2,
      pointBorderColor: "#fff",
      pointBackgroundColor: "#1e90ff",
      pointHoverBackgroundColor: "#ff6347",
      borderWidth: 3,
    },
  ],
});

const rateChartData = ref({
  labels: [],
  datasets: [
    {
      label: "Taxa de transmissão (bytes/s)",
      data: [],
      borderColor: "#ff6347",
      backgroundColor: null,
      fill: true,
      tension: 0.4,
      pointRadius: 5,
      pointHoverRadius: 8,
      pointBorderWidth: 2,
      pointBorderColor: "#fff",
      pointBackgroundColor: "#ff6347",
      pointHoverBackgroundColor: "#1e90ff",
      borderWidth: 3,
    },
  ],
});

const chartOptions = {
  responsive: true,
  animation: {
    duration: 800,
    easing: "easeInOutCubic",
  },
  interaction: {
    mode: "nearest",
    intersect: false,
  },
  plugins: {
    tooltip: {
      backgroundColor: "rgba(0, 0, 0, 0.85)",
      titleFont: { size: 15, weight: "bold" },
      bodyFont: { size: 14 },
      padding: 12,
      cornerRadius: 8,
      displayColors: false,
      callbacks: {
        title(context) {
          return `⏰ ${context[0].label}`;
        },
        label(context) {
          return selectedChart.value === "rate"
            ? `📦 ${context.parsed.y.toLocaleString()} bytes/s`
            : `📦 ${context.parsed.y.toLocaleString()} bytes`;
        },
      },
    },
    legend: {
      labels: {
        font: {
          size: 14,
          weight: "bold",
        },
        color: "#333",
        usePointStyle: true,
        pointStyle: "circle",
      },
    },
  },
  scales: {
    y: {
      beginAtZero: true,
      suggestedMax: suggestedMax.value,
      grid: {
        color: "rgba(0,0,0,0.05)",
        borderDash: [5, 5],
      },
      ticks: {
        color: "#444",
        font: {
          size: 13,
        },
        callback(value) {
          const formattedValue = value.toLocaleString();
          if (selectedChart.value === "rate") {
            return `${formattedValue} bytes/s`;
          } else {
            return `${formattedValue} bytes`;
          }
        },
      },
    },
    x: {
      grid: {
        display: false,
      },
      ticks: {
        color: "#444",
        font: {
          size: 13,
        },
      },
    },
  },
};

function addData(label, data) {
  const labels = [...toRaw(chartData.value.labels)];
  const datasetData = [...toRaw(chartData.value.datasets[0].data)];

  // Mantém no máximo 20 pontos
  if (labels.length >= historySize.value) {
    labels.shift();
    datasetData.shift();
  }

  labels.push(label);
  datasetData.push(data);

  chartData.value = {
    labels,
    datasets: [
      {
        ...chartData.value.datasets[0],
        data: datasetData,
      },
    ],
  };

  // Atualiza o gráfico de taxa de transmissão
  const rateLabels = [...toRaw(rateChartData.value.labels)];
  const rateData = [...toRaw(rateChartData.value.datasets[0].data)];

  if (rateLabels.length >= historySize.value) {
    rateLabels.shift();
    rateData.shift();
  }

  const recalculatedRate =
    rateLabels.length === 0 ? 0 : (data - previousData.value) / updateInterval.value; // Inicia com 0
  rateLabels.push(label);
  rateData.push(recalculatedRate);

  rateChartData.value = {
    labels: rateLabels,
    datasets: [
      {
        ...rateChartData.value.datasets[0],
        data: rateData,
      },
    ],
  };

  previousData.value = data; // Atualiza o dado anterior
}

async function fetchData() {
  try {
    const res = await axios.get("http://localhost:3001/snmp/trafego");
    const now = new Date();
    previousData.value = chartData.value.datasets[0].data.slice(-1)[0] || 0;
    addData(now.toLocaleTimeString(), res.data.bytes);
    lastUpdate.value = now.toLocaleTimeString();
    isConnected.value = true;
  } catch (e) {
    console.error("Erro ao buscar dados SNMP", e);
    const now = new Date();
    addData(now.toLocaleTimeString(), 0); // Adiciona 0 em caso de erro
    lastUpdate.value = `Erro: ${now.toLocaleTimeString()}`;
    isConnected.value = false;
  }
}

function startFetchingData() {
  if (updateIntervalId) clearInterval(updateIntervalId);
  updateIntervalId = setInterval(fetchData, updateInterval.value * 1000);
}

watch(updateInterval, () => {
  startFetchingData();
});

onMounted(() => {
  const canvas = document.querySelector("canvas");
  const ctx = canvas.getContext("2d");
  const gradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
  gradient.addColorStop(0, "rgba(30, 144, 255, 0.4)");
  gradient.addColorStop(1, "rgba(30, 144, 255, 0)");

  chartData.value.datasets[0].backgroundColor = gradient;

  const rateGradient = ctx.createLinearGradient(0, 0, 0, canvas.height);
  rateGradient.addColorStop(0, "rgba(255, 99, 71, 0.4)");
  rateGradient.addColorStop(1, "rgba(255, 99, 71, 0)");

  rateChartData.value.datasets[0].backgroundColor = rateGradient;

  fetchData();
  startFetchingData();
});
</script>

<script>
export default {
  components: {
    LineChart: Line,
    RateChart: Line,
  },
};
</script>

<style scoped>
.chart-container {
  max-width: 60%;
  height: 650px;
  margin: 20px auto;
  background-color: #ffffff;
  box-shadow: 0 6px 25px rgba(0, 0, 0, 0.08);
  border-radius: 18px;
  padding: 20px;
  box-sizing: border-box;
  font-family: "Segoe UI", system-ui, -apple-system, sans-serif;
  display: flex;
  flex-direction: column;
  border: 1px solid #eaeaea;
  transition: all 0.3s ease;
}

.chart-container:hover {
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.12);
}

.header {
  display: flex;
  justify-content: center; /* Centraliza horizontalmente */
  align-items: center; /* Centraliza verticalmente */
  margin-bottom: 20px; /* Espaçamento maior abaixo do cabeçalho */
  padding: 10px 0; /* Adiciona espaçamento interno */
  background-color: #f0f8ff; /* Fundo leve */
  border-radius: 12px; /* Bordas arredondadas */
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1); /* Sombra suave */
}

.title {
  color: #1e90ff; /* Azul vibrante */
  font-size: 24px; /* Tamanho maior */
  font-weight: 700; /* Negrito */
  margin: 0; /* Remove margens padrão */
  font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif; /* Fonte consistente */
  text-align: center; /* Centraliza o texto */
}

.controls {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15px;
  gap: 20px;
}

.stats {
  background: #f8f9fa;
  padding: 8px 15px;
  border-radius: 12px;
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.05);
}

.stat {
  display: flex;
  align-items: center;
  gap: 8px;
}

.stat-label {
  color: #6c757d;
  font-size: 13px;
  font-weight: 500;
}

.stat-value {
  color: #1e90ff;
  font-size: 14px;
  font-weight: 600;
}

.chart-wrapper {
  flex: 1;
  position: relative;
  width: 100%;
  height: 400px; /* Altura padrão */
}

@media (max-width: 1024px) {
  .chart-wrapper {
    height: 300px; /* Reduz altura em telas médias */
  }
}

@media (max-width: 768px) {
  .chart-wrapper {
    height: 250px; /* Reduz altura em telas pequenas */
  }
}

@media (max-width: 480px) {
  .chart-wrapper {
    height: 200px; /* Reduz altura em telas muito pequenas */
  }
}

canvas {
  width: 100% !important;
  height: 100% !important;
  transition: opacity 0.3s ease;
}

canvas:hover {
  opacity: 0.95;
}

.footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 10px;
  font-size: 13px;
  color: #6c757d;
}

.update-info {
  font-style: italic;
}

.connection-status {
  padding: 4px 10px;
  border-radius: 12px;
  font-weight: 500;
  font-size: 12px;
}

.connection-status.connected {
  background-color: rgba(46, 213, 115, 0.15);
  color: #2ed573;
}

.connection-status:not(.connected) {
  background-color: rgba(255, 71, 87, 0.15);
  color: #ff4757;
}

.update-interval {
  display: flex;
  align-items: center;
  gap: 8px;
}

#interval-input {
  width: 80px;
  padding: 6px 10px;
  border: 1px solid #ddd;
  border-radius: 20px;
  font-size: 14px;
  color: #333;
  background-color: #f9f9f9;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
}

#interval-input:focus {
  outline: none;
  border-color: #1e90ff;
  background-color: #ffffff;
  box-shadow: 0 0 8px rgba(30, 144, 255, 0.4);
}

#interval-input:hover {
  background-color: #ffffff;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.15);
}

.chart-selection {
  display: flex;
  align-items: center;
  gap: 8px;
}

#chart-select {
  padding: 6px 10px;
  border: 1px solid #ddd;
  border-radius: 20px;
  font-size: 14px;
  color: #333;
  background-color: #f9f9f9;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
}

#chart-select:focus {
  outline: none;
  border-color: #1e90ff;
  background-color: #ffffff;
  box-shadow: 0 0 8px rgba(30, 144, 255, 0.4);
}

#chart-select:hover {
  background-color: #ffffff;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.15);
}

@media (max-width: 768px) {
  .chart-container {
    height: auto;
    padding: 15px;
  }

  .header {
    flex-direction: column;
    align-items: flex-start;
    gap: 10px;
  }

  .title {
    font-size: 20px;
  }
}
</style>
