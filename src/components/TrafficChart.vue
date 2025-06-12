<template>
  <div class="chart-container">
    <div class="header">
      <h2 class="title">📊 Monitoramento de Tráfego de Rede</h2>
    </div>
    <div class="controls">
      <!-- Removido o painel de taxa atual -->
      <div class="update-interval">
        <label for="ethernet-select">Selecione a interface Ethernet:</label>
        <select id="ethernet-select" v-model="selectedEthernet">
          <option v-for="port in ethernetPorts" :key="port" :value="port">
            Ethernet {{ port }}
          </option>
        </select>
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
      <button
        class="toggle-button"
        :class="{ stop: isFetching, resume: !isFetching }"
        @click="toggleFetchingData"
      >
        {{ isFetching ? "Parar" : "Retomar" }}
      </button>
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
  <div v-if="!isConnected" class="error-message">
    Erro ao buscar dados. Verifique a conexão com o Mikrotik.
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
const updateInterval = ref(5); // Valor inicial em segundos
const selectedChart = ref("received"); // Gráfico inicial
const selectedEthernet = ref(1); // Interface Ethernet inicial
const ethernetPorts = [1, 2, 3, 4]; // Interfaces Ethernet disponíveis
let updateIntervalId = null;
const historySize = ref(20); // Tamanho máximo do histórico de dados
const isFetching = ref(false); // Estado para controlar se o gráfico está sendo atualizado

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
      label: "Bytes recebidos (rxBytes)",
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
      pointHoverBackgroundColor: "#1e90ff", // Keep hover color blue
      borderWidth: 3,
    },
    {
      label: "Bytes transmitidos (txBytes)",
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
      pointHoverBackgroundColor: "#ff6347", // Keep hover color orange
      borderWidth: 3,
    },
  ],
});

const rateChartData = ref({
  labels: [],
  datasets: [
    {
      label: "Taxa de recebimento (rxBits/s)",
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
    {
      label: "Taxa de transmissão (txBits/s)",
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

function formatBytes(bytes) {
  if (bytes >= 1e9) return `${(bytes / 1e9).toFixed(2)} GB`;
  if (bytes >= 1e6) return `${(bytes / 1e6).toFixed(2)} MB`;
  if (bytes >= 1e3) return `${(bytes / 1e3).toFixed(2)} KB`;
  return `${bytes.toFixed(2)} B`;
}

const chartOptions = {
  responsive: true,
  animation: {
    duration: 1500,
    easing: "easeOutQuart",
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
          const value = context.parsed.y;
          return selectedChart.value === "rate"
            ? `📦 ${formatBits(value)}`
            : `📦 ${formatBytes(value)}`;
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
          return selectedChart.value === "rate"
            ? formatBits(value)
            : formatBytes(value);
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

function formatBits(bits) {
  if (bits >= 1e9) return `${(bits / 1e9).toFixed(2)} Gb/s`;
  if (bits >= 1e6) return `${(bits / 1e6).toFixed(2)} Mb/s`;
  if (bits >= 1e3) return `${(bits / 1e3).toFixed(2)} kb/s`;
  return `${bits.toFixed(2)} b/s`;
}

function addData(label, rxBytes, txBytes) {
  const labels = [...toRaw(chartData.value.labels)];
  const rxData = [...toRaw(chartData.value.datasets[0].data)];
  const txData = [...toRaw(chartData.value.datasets[1].data)];

  if (labels.length >= historySize.value) {
    labels.shift();
    rxData.shift();
    txData.shift();
  }

  labels.push(label);
  rxData.push(rxBytes);
  txData.push(txBytes);

  chartData.value = {
    labels,
    datasets: [
      { ...chartData.value.datasets[0], data: rxData },
      { ...chartData.value.datasets[1], data: txData },
    ],
  };

  // Atualiza o gráfico de taxa de transmissão
  const rateLabels = [...toRaw(rateChartData.value.labels)];
  const rxRateData = [...toRaw(rateChartData.value.datasets[0].data)];
  const txRateData = [...toRaw(rateChartData.value.datasets[1].data)];

  if (rateLabels.length >= historySize.value) {
    rateLabels.shift();
    rxRateData.shift();
    txRateData.shift();
  }

  const rxRate =
    rxData.length > 1
      ? ((rxData[rxData.length - 1] - rxData[rxData.length - 2]) * 8) /
        updateInterval.value
      : 0; // Convertendo para bits
  const txRate =
    txData.length > 1
      ? ((txData[txData.length - 1] - txData[txData.length - 2]) * 8) /
        updateInterval.value
      : 0; // Convertendo para bits

  rateLabels.push(label);
  rxRateData.push(rxRate);
  txRateData.push(txRate);

  rateChartData.value = {
    labels: rateLabels,
    datasets: [
      { ...rateChartData.value.datasets[0], data: rxRateData },
      { ...rateChartData.value.datasets[1], data: txRateData },
    ],
  };
}

async function fetchData() {
  try {
    const res = await axios.get(
      `http://localhost:3001/snmp/trafego?rxPorta=${selectedEthernet.value}&txPorta=${selectedEthernet.value}`
    );
    const now = new Date();
    const { rxBytes, txBytes } = res.data.valores;

    addData(now.toLocaleTimeString(), rxBytes, txBytes);
    lastUpdate.value = now.toLocaleTimeString();
    isConnected.value = true;
    isFetching.value = true;
  } catch (e) {
    console.error("Erro ao buscar dados SNMP", e);
    isConnected.value = false;
    isFetching.value = false;

    lastUpdate.value = `Erro: ${new Date().toLocaleTimeString()}`;
  }
}

function startFetchingData() {
  if (updateIntervalId) clearInterval(updateIntervalId);
  updateIntervalId = setInterval(fetchData, updateInterval.value * 1000);
  console.log("Atualização do gráfico retomada.");
}

function stopFetchingData() {
  if (updateIntervalId) {
    clearInterval(updateIntervalId);
    updateIntervalId = null;
    console.log("Atualização do gráfico parada.");
  }
}

async function toggleFetchingData() {
  if (isFetching.value) {
    stopFetchingData();
    isConnected.value = false; // Atualiza o status para "Desconectado"
  } else {
    try {
      await fetchData(); // Tenta buscar os dados imediatamente
      startFetchingData(); // Se bem-sucedido, inicia o intervalo
      isConnected.value = true; // Atualiza o status para "Conectado"
    } catch (e) {
      console.error("Falha ao retomar a conexão", e);
      isConnected.value = false; // Atualiza o status para "Desconectado"
      return; // Sai da função sem alternar o estado
    }
  }
  isFetching.value = !isFetching.value; // Alterna o estado
}

watch([updateInterval, selectedEthernet], () => {
  startFetchingData();
});

onMounted(() => {
  const canvas = document.querySelector("canvas");
  if (!canvas) {
    console.error("Canvas não encontrado!");
    return;
  }

  const ctx = canvas.getContext("2d");
  if (!ctx) {
    console.error("Contexto do canvas não inicializado!");
    return;
  }

  // Gradiente para Bytes Recebidos
  const gradientRx = ctx.createLinearGradient(0, 0, 0, canvas.height || 400); // Garantir altura padrão
  gradientRx.addColorStop(0, "rgba(30, 144, 255, 0.4)");
  gradientRx.addColorStop(1, "rgba(30, 144, 255, 0)");

  chartData.value.datasets[0].backgroundColor = gradientRx;

  // Gradiente para Bytes Transmitidos
  const gradientTx = ctx.createLinearGradient(0, 0, 0, canvas.height || 400); // Garantir altura padrão
  gradientTx.addColorStop(0, "rgba(255, 99, 71, 0.4)"); // Laranja
  gradientTx.addColorStop(1, "rgba(255, 99, 71, 0)");

  chartData.value.datasets[1].backgroundColor = gradientTx;

  // Gradiente para Taxa de Transmissão (rxBytes/s)
  const rateGradientRx = ctx.createLinearGradient(
    0,
    0,
    0,
    canvas.height || 400
  );
  rateGradientRx.addColorStop(0, "rgba(30, 144, 255, 0.4)");
  rateGradientRx.addColorStop(1, "rgba(30, 144, 255, 0)");

  rateChartData.value.datasets[0].backgroundColor = rateGradientRx;

  // Gradiente para Taxa de Transmissão (txBytes/s)
  const rateGradientTx = ctx.createLinearGradient(
    0,
    0,
    0,
    canvas.height || 400
  );
  rateGradientTx.addColorStop(0, "rgba(255, 99, 71, 0.4)");
  rateGradientTx.addColorStop(1, "rgba(255, 99, 71, 0)");

  rateChartData.value.datasets[1].backgroundColor = rateGradientTx;

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
  max-width: 100%; /* Garante que o contêiner não ultrapasse a largura da tela */
  overflow: hidden; /* Evita que o conteúdo ultrapasse os limites */
  margin: 30px auto;
  padding: 25px;
  box-sizing: border-box;
  font-family: "Segoe UI", system-ui, -apple-system, sans-serif;
  display: flex;
  flex-direction: column;
  border: 1px solid #ddd;
  transition: all 0.3s ease;
}

.chart-container:hover {
  box-shadow: 0 6px 25px rgba(0, 0, 0, 0.15);
}

.header {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-bottom: 20px;
  padding: 10px 0;
  background-color: #eaf4fc;
  border-radius: 8px;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
}

.title {
  color: #007bff;
  font-size: 26px;
  font-weight: 700;
  margin: 0;
  text-align: center;
}

.controls {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  gap: 15px;
}

.update-interval,
.chart-selection {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

label {
  font-size: 14px;
  font-weight: 600;
  color: #555;
}

input,
select {
  width: 100%;
  max-width: 200px;
  padding: 8px 12px;
  border: 1px solid #ccc;
  border-radius: 6px;
  font-size: 14px;
  color: #333;
  background-color: #fff;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
}

input:focus,
select:focus {
  outline: none;
  border-color: #007bff;
  box-shadow: 0 0 8px rgba(0, 123, 255, 0.3);
}

input:hover,
select:hover {
  background-color: #f8f9fa;
}

.chart-wrapper {
  flex: 1;
  position: relative;
  margin: 20px 0;
  width: 100%; /* Garante que o gráfico ocupe 100% da largura disponível */
  max-width: 97.5%; /* Evita que o gráfico ultrapasse os limites do contêiner */
  height: 60vh;
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  padding: 15px;
  display: flex;
  align-items: center;
  justify-content: center;
}

canvas {
  width: 100% !important;
  height: 100% !important;
  border-radius: 8px;
}

.footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 15px;
  font-size: 13px;
  color: #666;
}

.update-info {
  font-style: italic;
}

.connection-status {
  padding: 6px 12px;
  border-radius: 8px;
  font-weight: 600;
  font-size: 12px;
}

.connection-status.connected {
  background-color: rgba(40, 167, 69, 0.1);
  color: #28a745;
}

.connection-status:not(.connected) {
  background-color: rgba(220, 53, 69, 0.1);
  color: #dc3545;
}

.toggle-button {
  padding: 10px 20px;
  color: #fff;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.3s ease;
  align-self: flex-end;
}

.toggle-button.stop {
  background-color: #dc3545; /* Vermelho para "Parar" */
}

.toggle-button.stop:hover {
  background-color: #c82333;
}

.toggle-button.stop:active {
  background-color: #bd2130;
}

.toggle-button.resume {
  background-color: #28a745; /* Verde para "Retomar" */
}

.toggle-button.resume:hover {
  background-color: #218838;
}

.toggle-button.resume:active {
  background-color: #1e7e34;
}

@media (max-width: 768px) {
  .chart-container {
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

  .controls {
    flex-direction: column;
    align-items: flex-start;
    gap: 15px;
  }

  .error-message {
    color: #dc3545;
    font-weight: bold;
    margin-top: 10px;
  }

  input,
  select {
    max-width: 100%;
  }
}
</style>
