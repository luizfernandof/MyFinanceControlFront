<script setup>
import { ref, onMounted, watch, computed, nextTick } from 'vue';
import api from '../services/api';
import DoughnutChart from '../components/DoughnutChart.vue';

const now = new Date();
const selectedMonth = ref(now.getMonth() + 1);
const selectedYear = ref(now.getFullYear());
const loading = ref(false);
const chartKey = ref(0); // Para forçar o refresh do componente filho

const summary = ref({ totalIncome: 0, totalExpense: 0, balance: 0 });
const expensesByCategory = ref([]);

// Prepara os dados para o componente filho (Sincronizado com o Java DTO 'total')
const chartData = computed(() => {
  return {
    labels: expensesByCategory.value.map(c => c.category),
    datasets: [{
      data: expensesByCategory.value.map(c => Number(c.total) || 0),
      backgroundColor: ['#6366f1', '#10b981', '#f59e0b', '#ef4444', '#8b5cf6', '#ec4899', '#06b6d4'],
      borderWidth: 0,
      hoverOffset: 15
    }]
  };
});

async function fetchData() {
  loading.value = true;
  try {
    const params = { month: selectedMonth.value, year: selectedYear.value };
    const [resSummary, resExpenses] = await Promise.all([
      api.get('/dashboard/summary', { params }),
      api.get('/dashboard/expenses-by-category', { params })
    ]);

    summary.value = resSummary.data;
    expensesByCategory.value = resExpenses.data || [];
    
    // Incrementa a key para garantir que o componente filho remonte se os dados mudarem
    await nextTick();
    chartKey.value++;
  } catch (error) {
    console.error("Erro ao carregar dashboard:", error);
  } finally {
    loading.value = false;
  }
}

watch([selectedMonth, selectedYear], fetchData);
onMounted(fetchData);

const months = [
  { value: 1, label: 'Janeiro' }, { value: 2, label: 'Fevereiro' },
  { value: 3, label: 'Março' }, { value: 4, label: 'Abril' },
  { value: 5, label: 'Maio' }, { value: 6, label: 'Junho' },
  { value: 7, label: 'Julho' }, { value: 8, label: 'Agosto' },
  { value: 9, label: 'Setembro' }, { value: 10, label: 'Outubro' },
  { value: 11, label: 'Novembro' }, { value: 12, label: 'Dezembro' }
];
const years = Array.from({ length: 6 }, (_, i) => now.getFullYear() - i);
</script>

<template>
  <div class="p-6 max-w-7xl mx-auto min-h-screen">
    
    <div class="flex justify-between items-center mb-10">
      <h2 class="text-3xl font-black text-slate-800 tracking-tight italic">
        Dashboard: <span class="text-blue-600">{{ months[selectedMonth - 1]?.label }}</span>
      </h2>
      <div class="flex gap-3 bg-white p-2.5 rounded-2xl shadow-sm border border-slate-100 items-center">
        <select v-model="selectedMonth" class="bg-transparent text-sm font-bold px-3 outline-none border-none focus:ring-0 cursor-pointer">
          <option v-for="m in months" :key="m.value" :value="m.value">{{ m.label }}</option>
        </select>
        <div class="w-px bg-slate-200 h-6"></div>
        <select v-model="selectedYear" class="bg-transparent text-sm font-bold px-3 outline-none border-none focus:ring-0 cursor-pointer">
          <option v-for="y in years" :key="y" :value="y">{{ y }}</option>
        </select>
      </div>
    </div>

    <div class="grid grid-cols-1 md:grid-cols-3 gap-6 mb-10">
      <div class="bg-white p-8 rounded-[2.5rem] shadow-xl border border-white flex flex-col items-center">
        <span class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2">Entradas</span>
        <h3 class="text-3xl font-black text-emerald-500">R$ {{ (summary.totalIncome || 0).toFixed(2) }}</h3>
      </div>

      <div class="bg-white p-8 rounded-[2.5rem] shadow-xl border border-white flex flex-col items-center">
        <span class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2">Saídas</span>
        <h3 class="text-3xl font-black text-rose-500">R$ {{ (summary.totalExpense || 0).toFixed(2) }}</h3>
      </div>

      <div class="bg-white p-8 rounded-[2.5rem] shadow-xl border border-white flex flex-col items-center">
        <span class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2">Saldo</span>
        <h3 :class="summary.balance >= 0 ? 'text-blue-600' : 'text-rose-600'" class="text-3xl font-black">
          R$ {{ (summary.balance || 0).toFixed(2) }}
        </h3>
      </div>
    </div>

    <div class="bg-white p-12 rounded-[3.5rem] shadow-xl border border-white flex flex-col items-center">
      <h3 class="text-xs font-black text-slate-400 uppercase tracking-widest mb-10 text-center">Distribuição de Gastos</h3>
      
      <div class="w-full max-w-lg">
        <DoughnutChart 
          v-if="expensesByCategory.length > 0" 
          :key="chartKey"
          :chartData="chartData" 
        />
        <div v-else-if="!loading" class="text-center py-20 text-slate-300 italic">
          Nenhum registro encontrado para este período.
        </div>
        <div v-else class="text-center py-20">
          <div class="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-600 mx-auto"></div>
        </div>
      </div>
    </div>
  </div>
</template>