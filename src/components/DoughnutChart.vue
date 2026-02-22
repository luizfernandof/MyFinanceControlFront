<script setup>
import { Doughnut } from 'vue-chartjs';
import { 
  Chart as ChartJS, Title, Tooltip, Legend, ArcElement, CategoryScale 
} from 'chart.js';

ChartJS.register(Title, Tooltip, Legend, ArcElement, CategoryScale);

const props = defineProps({
  chartData: {
    type: Object,
    required: true
  }
});

const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  cutout: '70%',
  plugins: {
    legend: {
      position: 'bottom',
      labels: { padding: 20, font: { weight: 'bold', family: 'Inter' } }
    }
  }
};
</script>

<template>
  <div style="position: relative; height: 350px; width: 100%;">
    <Doughnut :data="props.chartData" :options="chartOptions" />
  </div>
</template>