<script setup>
import { ref, onMounted, watch } from 'vue';
import api from '../services/api';
import { useBreakpoint } from '../composables/useBreakpoint';

// Componentes
import TransactionTableDesktop from '../components/TransactionTableDesktop.vue';
import TransactionCardsMobile from '../components/TransactionCardsMobile.vue';
import TransactionModal from '../components/TransactionModal.vue';
import ConfirmModal from '../components/ConfirmModal.vue';

const { isMobile } = useBreakpoint();

// --- ESTADOS ---
const transactions = ref([]);
const categories = ref([]);
const selectedMonth = ref(new Date().getMonth() + 1);
const selectedYear = ref(new Date().getFullYear());

// PAGINAÇÃO
const currentPage = ref(0);
const pageSize = ref(10);
const totalElements = ref(0);
const totalPages = ref(0);

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

// --- LÓGICA DE DADOS ---
async function fetchData() {
  loading.value = true;
  try {
    const params = {
      month: selectedMonth.value,
      year: selectedYear.value,
      page: currentPage.value,
      size: pageSize.value,
      sort: 'date,desc'
    };
    const [resTrans, resCats] = await Promise.all([
      api.get('/transactions', { params }),
      api.get('/categories')
    ]);

    // Mapeamento dos dados
    transactions.value = resTrans.data.content || [];
    totalElements.value = Number(resTrans.data.totalElements) || 0;
    totalPages.value = Number(resTrans.data.totalPages) || 0;
    categories.value = resCats.data || [];
  } catch (e) {
    console.error("Erro ao carregar dados:", e);
  } finally {
    loading.value = false;
  }
}

// Watchers
watch([selectedMonth, selectedYear, pageSize], () => {
  currentPage.value = 0;
  fetchData();
});
watch(currentPage, fetchData);
watch(isMobile, (newVal) => {
  pageSize.value = newVal ? 1000 : 10;
}, { immediate: true });

// --- AÇÕES ---
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
    apiErrorMessage.value = error.response?.data?.message || "Erro no servidor.";
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

onMounted(fetchData);

const months = [
  { value: 1, label: 'Janeiro' }, { value: 2, label: 'Fevereiro' }, { value: 3, label: 'Março' },
  { value: 4, label: 'Abril' }, { value: 5, label: 'Maio' }, { value: 6, label: 'Junho' },
  { value: 7, label: 'Julho' }, { value: 8, label: 'Agosto' }, { value: 9, label: 'Setembro' },
  { value: 10, label: 'Outubro' }, { value: 11, label: 'Novembro' }, { value: 12, label: 'Dezembro' }
];
const years = Array.from({ length: 6 }, (_, i) => new Date().getFullYear() - i);
</script>

<template>
  <div class="p-3 md:p-6 max-w-7xl mx-auto min-h-screen flex flex-col">

    <div class="flex flex-col md:flex-row justify-between items-start md:items-center gap-4 mb-6 md:mb-10">
      <div class="w-full md:w-auto text-left">
        <h2 class="text-2xl md:text-3xl font-black text-slate-800 italic uppercase tracking-tighter leading-tight">
          Transações</h2>
        <p class="text-slate-400 text-[9px] font-black uppercase tracking-[0.2em]">{{ months[selectedMonth - 1].label }}
          {{ selectedYear }}</p>
      </div>

      <div class="flex flex-wrap items-center gap-2 w-full md:w-auto h-auto md:h-14">

        <div v-if="!isMobile"
          class="flex items-center bg-white p-1 rounded-xl md:rounded-2xl border border-slate-100 shadow-sm h-full">

          <button @click="currentPage--" :disabled="currentPage === 0"
            class="px-4 py-2 text-blue-600 disabled:text-slate-200 disabled:opacity-50 hover:scale-110 transition-all active:scale-95">
            <font-awesome-icon icon="arrow-left" class="text-lg md:text-xl" />
          </button>

          <div class="w-[1px] h-6 bg-slate-100"></div>

          <button @click="currentPage++" :disabled="transactions.length < pageSize"
            class="px-4 py-2 text-blue-600 disabled:text-slate-200 disabled:opacity-50 hover:scale-110 transition-all active:scale-95">
            <font-awesome-icon icon="arrow-right" class="text-lg md:text-xl" />
          </button>
        </div>

        <div v-if="!isMobile"
          class="flex items-center bg-white px-4 py-1.5 md:py-0 rounded-xl md:rounded-2xl border border-slate-100 shadow-sm h-full">
          <span class="text-[9px] font-black text-slate-400 uppercase tracking-widest mr-2">EXIBIR:</span>
          <select v-model="pageSize"
            class="bg-transparent text-[10px] md:text-xs font-black outline-none cursor-pointer text-blue-600 appearance-none min-w-[30px] text-center">
            <option :value="5">05</option>
            <option :value="10">10</option>
            <option :value="15">15</option>
            <option :value="30">30</option>
            <option :value="50">50</option>
          </select>
        </div>

        <div
          class="flex h-11 md:h-full gap-1 bg-white p-1.5 rounded-xl md:rounded-2xl shadow-sm border border-slate-100 flex-grow md:flex-grow-0 min-w-[140px]">
          <select v-model="selectedMonth"
            class="bg-transparent text-[10px] md:text-xs font-black outline-none w-full text-center appearance-none cursor-pointer">
            <option v-for="m in months" :key="m.value" :value="m.value">{{ m.label }}</option>
          </select>
          <div class="w-[1px] h-4 bg-slate-100 self-center"></div>
          <select v-model="selectedYear"
            class="bg-transparent text-[10px] md:text-xs font-black outline-none text-center appearance-none px-2 cursor-pointer">
            <option v-for="y in years" :key="y" :value="y">{{ y }}</option>
          </select>
        </div>

        <button @click="openCreate"
          class="h-11 md:h-full bg-blue-600 text-white px-6 rounded-xl md:rounded-[1.5rem] font-black shadow-lg shadow-blue-100 active:scale-95 transition-all text-[10px] md:text-xs uppercase flex items-center gap-2 flex-grow md:flex-grow-0">
          <font-awesome-icon icon="plus" />
          <span>NOVO</span>
        </button>
      </div>
    </div>

    <div class="flex-grow">
      <TransactionTableDesktop v-if="!isMobile" :transactions="transactions" @edit="prepareEdit"
        @delete="openDeleteConfirm" />
      <TransactionCardsMobile v-else :transactions="transactions" @edit="prepareEdit" @delete="openDeleteConfirm" />
    </div>

    <TransactionModal :show="showFormModal" :editing="isEditing" :categories="categories" :initialData="transactionForm"
      :apiError="apiErrorMessage" @close="showFormModal = false" @save="handleSave" />

    <ConfirmModal :show="showConfirmModal" title="Excluir Registro?"
      :message="`Deseja realmente excluir '${transactionToDelete?.description}'?`" confirmText="Sim, Excluir"
      @close="showConfirmModal = false" @confirm="confirmDelete" />

  </div>
</template>