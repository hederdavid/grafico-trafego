<template>
  <div class="container">
    <h1 class="title">📡 Monitoramento de Tráfego</h1>
    <TrafficChart />
  </div>
</template>

<script setup>
import TrafficChart from './components/TrafficChart.vue';
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding-top: 2rem;
  gap: 3rem;
  background: linear-gradient(to bottom right, #f0f8ff, #e6f2ff);
  min-height: 100vh;
}

.title {
  font-size: 2.5rem;
  font-weight: 700;
  color: #1e90ff;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}
</style>
