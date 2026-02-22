<script setup>
import { ref, onMounted, watch } from 'vue';
import api from '../services/api';
import TransactionModal from '../components/TransactionModal.vue';
import ConfirmModal from '../components/ConfirmModal.vue';

// --- ESTADOS ---
const selectedMonth = ref(new Date().getMonth() + 1);
const selectedYear = ref(new Date().getFullYear());
const transactions = ref([]);
const categories = ref([]);
const loading = ref(false);
const showFormModal = ref(false);
const isEditing = ref(false);
const apiErrorMessage = ref('');

const showConfirmModal = ref(false);
const transactionToDelete = ref(null);

const initialForm = {
  id: null, description: '', amount: '', date: new Date().toISOString().split('T')[0],
  type: '', categoryId: '', installments: 1, recurring: false, occurrences: 12
};
const transactionForm = ref({ ...initialForm });

// --- AUXILIARES ---
function formatDate(dateString) {
  const [year, month, day] = dateString.split('-');
  return `${day}/${month}/${year}`;
}

async function fetchData() {
  loading.value = true;
  try {
    const params = { 
      month: selectedMonth.value, 
      year: selectedYear.value, 
      page: 0, 
      size: 50, 
      sort: 'date,desc' 
    };
    const [resTrans, resCats] = await Promise.all([
      api.get('/transactions', { params }),
      api.get('/categories')
    ]);
    transactions.value = resTrans.data.content;
    categories.value = resCats.data;
  } catch (e) { 
    console.error(e); 
  } finally { 
    loading.value = false; 
  }
}

async function handleSave(payload) {
  apiErrorMessage.value = '';
  try {
    if (isEditing.value) {
      await api.put(`/transactions/${payload.id}`, payload);
    } else {
      await api.post('/transactions', payload);
    }
    showFormModal.value = false;
    fetchData();
  } catch (error) {
    apiErrorMessage.value = error.response?.data?.message || "Erro na validação do servidor.";
  }
}

function openDeleteConfirm(t) {
  transactionToDelete.value = t;
  showConfirmModal.value = true;
}

async function confirmDelete() {
  if (!transactionToDelete.value) return;
  try {
    await api.delete(`/transactions/${transactionToDelete.value.id}`);
    showConfirmModal.value = false;
    transactionToDelete.value = null;
    fetchData();
  } catch (e) { 
    console.error(e); 
  }
}

function openCreate() {
  isEditing.value = false;
  transactionForm.value = { ...initialForm };
  showFormModal.value = true;
}

function prepareEdit(t) {
  isEditing.value = true;
  const categoryId = categories.value.find(c => c.name === t.categoryName)?.id || '';
  transactionForm.value = { ...t, categoryId };
  showFormModal.value = true;
}

onMounted(fetchData);
watch([selectedMonth, selectedYear], fetchData);

const months = [
  { value: 1, label: 'Janeiro' }, { value: 2, label: 'Fevereiro' },
  { value: 3, label: 'Março' }, { value: 4, label: 'Abril' },
  { value: 5, label: 'Maio' }, { value: 6, label: 'Junho' },
  { value: 7, label: 'Julho' }, { value: 8, label: 'Agosto' },
  { value: 9, label: 'Setembro' }, { value: 10, label: 'Outubro' },
  { value: 11, label: 'Novembro' }, { value: 12, label: 'Dezembro' }
];
const years = Array.from({ length: 6 }, (_, i) => new Date().getFullYear() - i);
</script>

<template>
  <div class="p-6 max-w-7xl mx-auto min-h-screen">
    
    <div class="flex justify-between items-center mb-10">
      <div>
        <h2 class="text-3xl font-black text-slate-800 italic uppercase tracking-tighter">Transações</h2>
        <p class="text-slate-400 text-[10px] font-black uppercase tracking-[0.3em]">
          {{ months[selectedMonth-1].label }} {{ selectedYear }}
        </p>
      </div>
      
      <div class="flex gap-4">
        <div class="flex gap-2 bg-white p-2 rounded-2xl shadow-sm border border-slate-100">
           <select v-model="selectedMonth" class="bg-transparent text-xs font-black outline-none px-2 cursor-pointer">
              <option v-for="m in months" :key="m.value" :value="m.value">{{ m.label }}</option>
           </select>
           <select v-model="selectedYear" class="bg-transparent text-xs font-black outline-none px-2 cursor-pointer">
              <option v-for="y in years" :key="y" :value="y">{{ y }}</option>
           </select>
        </div>
        <button @click="openCreate" class="bg-blue-600 text-white px-8 py-4 rounded-[1.5rem] font-black shadow-xl shadow-blue-100 hover:scale-105 transition-all text-xs uppercase tracking-widest flex items-center gap-2">
          <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="3" d="M12 4v16m8-8H4"/></svg>
          Novo Lançamento
        </button>
      </div>
    </div>

    <div class="bg-white rounded-[2.5rem] shadow-xl overflow-hidden border border-slate-50">
      <table class="w-full text-left border-collapse">
        <thead class="bg-slate-50/50 text-[9px] font-black uppercase text-slate-400 tracking-[0.2em]">
          <tr>
            <th class="px-8 py-5">Data</th>
            <th class="px-8 py-5">Descrição</th>
            <th class="px-8 py-5">Categoria</th>
            <th class="px-8 py-5 text-right">Valor</th>
            <th class="px-8 py-5 text-right">Ações</th>
          </tr>
        </thead>
        <tbody class="divide-y divide-slate-50">
          <tr v-for="t in transactions" :key="t.id" class="hover:bg-slate-50/50 transition-colors group">
            <td class="px-8 py-5 text-xs font-bold text-slate-400 italic">
                {{ formatDate(t.date) }}
            </td>
            <td class="px-8 py-5 text-sm font-black text-slate-700">
              <div class="flex items-center gap-2">
                {{ t.description }}
                <span v-if="t.groupId" class="w-2 h-2 rounded-full bg-blue-400 animate-pulse" title="Lançamento Recorrente"></span>
              </div>
            </td>
            
            <td class="px-8 py-5">
              <span 
                :class="t.type === 'INCOME' 
                  ? 'bg-emerald-50 text-emerald-600 border-emerald-100' 
                  : 'bg-rose-50 text-rose-600 border-rose-100'"
                class="px-3 py-1.5 rounded-full font-black uppercase tracking-tighter border text-[10px]"
              >
                {{ t.categoryName }}
              </span>
            </td>

            <td class="px-8 py-5 text-sm font-black text-right tracking-tight" :class="t.type === 'INCOME' ? 'text-emerald-500' : 'text-rose-500'">
              {{ t.type === 'INCOME' ? '+' : '-' }} R$ {{ (t.amount || 0).toFixed(2) }}
            </td>
            
            <td class="px-8 py-5 text-right">
              <div class="flex justify-end gap-6">
                <button @click="prepareEdit(t)" class="flex items-center gap-1 text-[10px] font-black text-blue-600 hover:text-blue-800 uppercase tracking-widest transition-colors">
                  <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15.232 5.232l3.536 3.536m-2.036-5.036a2.5 2.5 0 113.536 3.536L6.5 21.036H3v-3.572L16.732 3.732z"/></svg>
                  Editar
                </button>
                <button @click="openDeleteConfirm(t)" class="flex items-center gap-1 text-[10px] font-black text-rose-300 hover:text-rose-600 uppercase tracking-widest transition-colors">
                  <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"/></svg>
                  Excluir
                </button>
              </div>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <TransactionModal 
      :show="showFormModal"
      :editing="isEditing"
      :categories="categories"
      :initialData="transactionForm"
      :apiError="apiErrorMessage"
      @close="showFormModal = false"
      @save="handleSave"
    />

    <ConfirmModal 
      :show="showConfirmModal"
      title="Excluir Registro?"
      :message="`Deseja realmente apagar '${transactionToDelete?.description}'?`"
      confirmText="Sim, Excluir"
      @close="showConfirmModal = false"
      @confirm="confirmDelete"
    />
  </div>
</template>