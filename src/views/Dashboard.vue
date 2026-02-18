<script setup>
import { ref, onMounted, watch, computed } from 'vue';
import api from '../services/api';

const now = new Date();

// --- ESTADOS ---
const selectedMonth = ref(now.getMonth() + 1);
const selectedYear = ref(now.getFullYear());
const loading = ref(false);

const summary = ref({
  totalIncome: 0,
  totalExpense: 0,
  balance: 0
});

const allTransactions = ref([]);

// --- OPÇÕES DOS SELECTS ---
const months = [
  { value: 1, label: 'Janeiro' }, { value: 2, label: 'Fevereiro' },
  { value: 3, label: 'Março' }, { value: 4, label: 'Abril' },
  { value: 5, label: 'Maio' }, { value: 6, label: 'Junho' },
  { value: 7, label: 'Julho' }, { value: 8, label: 'Agosto' },
  { value: 9, label: 'Setembro' }, { value: 10, label: 'Outubro' },
  { value: 11, label: 'Novembro' }, { value: 12, label: 'Dezembro' }
];
const years = Array.from({ length: 6 }, (_, i) => now.getFullYear() - i);

// --- LÓGICA DE FILTRO (VISUALIZADOR) ---
// Filtra a lista de transações no front-end para garantir que a tabela mude
const filteredTransactions = computed(() => {
  return allTransactions.value.filter(t => {
    const d = new Date(t.date);
    // Usamos getUTC para evitar que fusos horários mudem o dia/mês da transação
    const transMonth = d.getUTCMonth() + 1;
    const transYear = d.getUTCFullYear();
    return transMonth === selectedMonth.value && transYear === selectedYear.value;
  });
});

// --- BUSCA DE DADOS ---
async function fetchData() {
  loading.value = true;
  try {
    const params = {
      month: selectedMonth.value,
      year: selectedYear.value
    };

    // Buscamos o resumo (filtrado pela API) e a lista (que filtramos no computed)
    const [resSummary, resTransactions] = await Promise.all([
      api.get('/dashboard/summary', { params }),
      api.get('/transactions') 
    ]);

    summary.value = {
      totalIncome: resSummary.data?.totalIncome ?? 0,
      totalExpense: resSummary.data?.totalExpense ?? 0,
      balance: resSummary.data?.balance ?? 0
    };

    allTransactions.value = resTransactions.data ?? [];
  } catch (error) {
    console.error("Erro ao carregar dados do dashboard:", error);
  } finally {
    loading.value = false;
  }
}

// Reage à mudança dos selects
watch([selectedMonth, selectedYear], () => {
  fetchData();
});

onMounted(fetchData);
</script>

<template>
  <div class="p-6 max-w-7xl mx-auto min-h-screen">
    
    <div class="flex flex-col md:flex-row justify-between items-start md:items-center mb-8 gap-4">
      <div :class="{ 'opacity-40': loading }" class="transition-opacity duration-300">
        <h2 class="text-2xl font-black text-slate-800 tracking-tight">
          Dashboard: <span class="text-blue-600">{{ months[selectedMonth - 1]?.label }}</span>
        </h2>
        <p class="text-slate-400 text-sm font-medium italic">Visualizando movimentações de {{ selectedYear }}</p>
      </div>
      
      <div class="flex gap-3 bg-white p-2 rounded-2xl shadow-sm border border-slate-100 items-center">
        <select v-model="selectedMonth" 
          class="bg-transparent text-slate-600 text-sm font-bold px-3 py-1 outline-none cursor-pointer hover:text-blue-600">
          <option v-for="m in months" :key="m.value" :value="m.value">{{ m.label }}</option>
        </select>
        <div class="w-px bg-slate-200 h-6"></div>
        <select v-model="selectedYear" 
          class="bg-transparent text-slate-600 text-sm font-bold px-3 py-1 outline-none cursor-pointer hover:text-blue-600">
          <option v-for="y in years" :key="y" :value="y">{{ y }}</option>
        </select>
        <div v-if="loading" class="ml-2 animate-spin h-4 w-4 border-2 border-blue-600 border-t-transparent rounded-full"></div>
      </div>
    </div>

    <div class="grid grid-cols-1 md:grid-cols-3 gap-6 mb-12">
      <div class="bg-white p-8 rounded-[2rem] shadow-xl shadow-slate-200/50 border border-white flex flex-col items-center">
        <div class="w-10 h-10 bg-emerald-50 text-emerald-500 rounded-full flex items-center justify-center mb-3">
          <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 7h8m0 0v8m0-8l-8 8-4-4-6 6"></path></svg>
        </div>
        <span class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">Entradas</span>
        <h3 class="text-2xl font-black text-emerald-500 tracking-tighter">R$ {{ (summary.totalIncome || 0).toFixed(2) }}</h3>
      </div>

      <div class="bg-white p-8 rounded-[2rem] shadow-xl shadow-slate-200/50 border border-white flex flex-col items-center">
        <div class="w-10 h-10 bg-rose-50 text-rose-500 rounded-full flex items-center justify-center mb-3">
          <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 17h8m0 0v-8m0 8l-8-8-4 4-6-6"></path></svg>
        </div>
        <span class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">Saídas</span>
        <h3 class="text-2xl font-black text-rose-500 tracking-tighter">R$ {{ (summary.totalExpense || 0).toFixed(2) }}</h3>
      </div>

      <div class="bg-white p-8 rounded-[2rem] shadow-xl shadow-slate-200/50 border border-white flex flex-col items-center">
        <div class="w-10 h-10 bg-blue-50 text-blue-500 rounded-full flex items-center justify-center mb-3">
          <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8c-1.657 0-3 .895-3 2s1.343 2 3 2 3 .895 3 2-1.343 2-3 2m0-8c1.11 0 2.08.402 2.599 1M12 8V7m0 1v8m0 0v1m0-1c-1.11 0-2.08-.402-2.599-1M21 12a9 9 0 11-18 0 9 9 0 0118 0z"></path></svg>
        </div>
        <span class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">Saldo</span>
        <h3 :class="(summary.balance || 0) >= 0 ? 'text-blue-600' : 'text-rose-600'" class="text-2xl font-black tracking-tighter">
          R$ {{ (summary.balance || 0).toFixed(2) }}
        </h3>
      </div>
    </div>

    <div class="bg-white rounded-[2.5rem] shadow-sm border border-slate-100 overflow-hidden">
        <div class="p-8 border-b border-slate-50 flex justify-between items-center bg-slate-50/30">
            <h3 class="font-black text-slate-800 uppercase text-xs tracking-widest">Lançamentos de {{ months[selectedMonth - 1]?.label }}</h3>
            <span class="text-[10px] bg-blue-600 px-3 py-1 rounded-full font-bold text-white shadow-sm shadow-blue-200">
                {{ filteredTransactions.length }} itens
            </span>
        </div>

        <div v-if="filteredTransactions.length > 0" class="overflow-x-auto">
            <table class="w-full text-left">
                <thead class="bg-slate-50/50 text-slate-400">
                    <tr>
                        <th class="px-8 py-4 text-[10px] font-black uppercase">Data</th>
                        <th class="px-8 py-4 text-[10px] font-black uppercase">Descrição</th>
                        <th class="px-8 py-4 text-[10px] font-black uppercase text-right">Valor</th>
                    </tr>
                </thead>
                <tbody class="divide-y divide-slate-50">
                    <tr v-for="t in filteredTransactions" :key="t.id" class="hover:bg-slate-50/50 transition-colors group">
                        <td class="px-8 py-4 text-xs font-medium text-slate-400">
                            {{ new Date(t.date).toLocaleDateString('pt-BR', { timeZone: 'UTC' }) }}
                        </td>
                        <td class="px-8 py-4">
                            <span class="block text-sm font-bold text-slate-700 group-hover:text-blue-600 transition-colors">{{ t.description }}</span>
                            <span class="text-[10px] font-bold text-slate-300 uppercase tracking-tighter">{{ t.categoryName }}</span>
                        </td>
                        <td class="px-8 py-4 text-sm font-black text-right" :class="t.type === 'INCOME' ? 'text-emerald-500' : 'text-rose-500'">
                            {{ t.type === 'INCOME' ? '+' : '-' }} R$ {{ (t.amount || 0).toFixed(2) }}
                        </td>
                    </tr>
                </tbody>
            </table>
        </div>

        <div v-else class="p-24 text-center">
            <div class="text-slate-200 mb-4">
              <svg class="w-12 h-12 mx-auto" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2"></path></svg>
            </div>
            <p class="text-slate-400 italic text-sm">Nenhum registro encontrado para este período.</p>
        </div>
    </div>
  </div>
</template>