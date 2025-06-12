<template>
  <div class="container">
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
</style>
