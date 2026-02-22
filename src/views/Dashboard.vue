<script setup>
import { ref, onMounted, watch, computed, nextTick } from 'vue';
import api from '../services/api';
import DoughnutChart from '../components/DoughnutChart.vue';
import { useBreakpoint } from '../composables/useBreakpoint';

const { isMobile } = useBreakpoint();
const now = new Date();
const selectedMonth = ref(now.getMonth() + 1);
const selectedYear = ref(now.getFullYear());
const loading = ref(false);
const chartKey = ref(0);

const summary = ref({ totalIncome: 0, totalExpense: 0, balance: 0 });
const expensesByCategory = ref([]);

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
  <div class="p-4 md:p-6 max-w-7xl mx-auto min-h-screen">
    
    <div class="flex flex-col md:flex-row justify-between items-start md:items-center gap-4 mb-8 md:mb-10">
      <div class="w-full md:w-auto">
        <h2 class="text-2xl md:text-3xl font-black text-slate-800 italic uppercase tracking-tighter leading-tight">
          Dashboard
        </h2>
        <p class="text-slate-400 text-[9px] font-black uppercase tracking-[0.2em]">
          {{ months[selectedMonth - 1]?.label }} {{ selectedYear }}
        </p>
      </div>

      <div class="flex h-11 md:h-14 gap-1 bg-white p-1.5 rounded-xl md:rounded-2xl shadow-sm border border-slate-100 w-full md:w-auto">
        <select v-model="selectedMonth" class="bg-transparent text-[10px] md:text-xs font-black outline-none w-full text-center appearance-none cursor-pointer">
          <option v-for="m in months" :key="m.value" :value="m.value">{{ m.label }}</option>
        </select>
        <div class="w-[1px] h-4 bg-slate-100 self-center"></div>
        <select v-model="selectedYear" class="bg-transparent text-[10px] md:text-xs font-black outline-none px-2 appearance-none text-center cursor-pointer">
          <option v-for="y in years" :key="y" :value="y">{{ y }}</option>
        </select>
      </div>
    </div>

    <div class="grid grid-cols-1 md:grid-cols-3 gap-4 md:gap-6 mb-8 md:mb-10">
      <div class="bg-white p-6 md:p-8 rounded-[1.5rem] md:rounded-[2.5rem] shadow-xl border border-white flex flex-col items-center">
        <span class="text-[9px] md:text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1 md:mb-2">Entradas</span>
        <h3 class="text-2xl md:text-3xl font-black text-emerald-500">R$ {{ (summary.totalIncome || 0).toFixed(2) }}</h3>
      </div>

      <div class="bg-white p-6 md:p-8 rounded-[1.5rem] md:rounded-[2.5rem] shadow-xl border border-white flex flex-col items-center">
        <span class="text-[9px] md:text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1 md:mb-2">Saídas</span>
        <h3 class="text-2xl md:text-3xl font-black text-rose-500">R$ {{ (summary.totalExpense || 0).toFixed(2) }}</h3>
      </div>

      <div class="bg-white p-6 md:p-8 rounded-[1.5rem] md:rounded-[2.5rem] shadow-xl border border-white flex flex-col items-center">
        <span class="text-[9px] md:text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1 md:mb-2">Saldo</span>
        <h3 :class="summary.balance >= 0 ? 'text-blue-600' : 'text-rose-600'" class="text-2xl md:text-3xl font-black">
          R$ {{ (summary.balance || 0).toFixed(2) }}
        </h3>
      </div>
    </div>

    <div class="bg-white p-6 md:p-12 rounded-[2rem] md:rounded-[3.5rem] shadow-xl border border-white flex flex-col items-center">
      <h3 class="text-[10px] md:text-xs font-black text-slate-400 uppercase tracking-widest mb-6 md:mb-10 text-center italic">
        Distribuição de Gastos
      </h3>
      
      <div class="w-full" :class="isMobile ? 'max-w-[280px]' : 'max-w-lg'">
        <DoughnutChart 
          v-if="expensesByCategory.length > 0" 
          :key="chartKey"
          :chartData="chartData" 
        />
        <div v-else-if="!loading" class="text-center py-10 md:py-20 text-slate-300 italic text-sm">
          Nenhum registro encontrado para este período.
        </div>
        <div v-else class="text-center py-10 md:py-20">
          <div class="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-600 mx-auto"></div>
        </div>
      </div>
    </div>
  </div>
</template>