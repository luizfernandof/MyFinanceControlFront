<script setup>
import { ref, onMounted } from 'vue';
import api from '../services/api';
import { useBreakpoint } from '../composables/useBreakpoint';
import BaseInput from '../components/BaseInput.vue';
import ConfirmModal from '../components/ConfirmModal.vue';

const { isMobile } = useBreakpoint();

// --- ESTADOS ---
const categories = ref([]);
const loading = ref(false);
const apiErrorMessage = ref('');
const errors = ref({ name: '' });

const showFormModal = ref(false);
const showDeleteConfirm = ref(false);
const showNoticeModal = ref(false);

const categoryForm = ref({ id: null, name: '', type: 'EXPENSE' });
const isEditing = ref(false);
const categoryToDelete = ref(null);

// --- VALIDAÇÃO ---
function validateForm() {
  errors.value.name = '';
  if (!categoryForm.value.name.trim()) {
    errors.value.name = 'O nome da categoria é obrigatório.';
    return false;
  }
  return true;
}

// --- CARREGAMENTO ---
async function fetchCategories() {
  loading.value = true;
  try {
    const response = await api.get('/categories');
    categories.value = response.data;
  } catch (error) {
    showError("Erro ao carregar categorias");
  } finally {
    loading.value = false;
  }
}

// --- SALVAR ---
async function saveCategory() {
  if (!validateForm()) return;
  try {
    if (isEditing.value) {
      await api.put(`/categories/${categoryForm.value.id}`, categoryForm.value);
    } else {
      await api.post('/categories', categoryForm.value);
    }
    showFormModal.value = false;
    resetForm();
    fetchCategories();
  } catch (error) {
    const msg = error.response?.data?.message || "Erro inesperado ao processar.";
    showError(msg);
  }
}

// --- EXCLUSÃO ---
function openDeleteConfirm(cat) {
  categoryToDelete.value = cat;
  showDeleteConfirm.value = true;
}

async function confirmDelete() {
  if (categoryToDelete.value) {
    try {
      await api.delete(`/categories/${categoryToDelete.value.id}`);
      showDeleteConfirm.value = false;
      fetchCategories();
    } catch (error) {
      showError("Não foi possível excluir. Verifique se existem transações vinculadas.");
    }
  }
}

// --- AUXILIARES ---
function openCreateModal() { resetForm(); showFormModal.value = true; }
function prepareEdit(category) {
  resetForm();
  categoryForm.value = { ...category };
  isEditing.value = true;
  showFormModal.value = true;
}
function resetForm() {
  categoryForm.value = { id: null, name: '', type: 'EXPENSE' };
  isEditing.value = false;
  apiErrorMessage.value = '';
  errors.value.name = '';
}
function showError(msg) { apiErrorMessage.value = msg; showNoticeModal.value = true; }

onMounted(fetchCategories);
</script>

<template>
  <div class="p-4 md:p-6 max-w-5xl mx-auto min-h-screen">
    
    <div class="flex flex-col md:flex-row justify-between items-start md:items-center gap-4 mb-8 md:mb-10">
      <div class="w-full md:w-auto">
        <h2 class="text-2xl md:text-3xl font-black text-slate-800 italic uppercase tracking-tighter leading-tight">Categorias</h2>
        <p class="text-slate-400 text-[9px] font-black uppercase tracking-[0.2em]">Gerencie seus grupos de custo</p>
      </div>

      <button @click="openCreateModal" 
        class="h-11 md:h-14 bg-blue-600 text-white px-6 md:px-8 rounded-xl md:rounded-[1.5rem] font-black shadow-lg shadow-blue-100 hover:scale-105 active:scale-95 transition-all text-[10px] md:text-xs uppercase tracking-widest flex items-center justify-center gap-2 w-full md:w-auto">
        <font-awesome-icon icon="fa-solid fa-plus" />
        <span>NOVA CATEGORIA</span>
      </button>
    </div>

    <div v-if="!isMobile" class="bg-white rounded-[2.5rem] shadow-xl overflow-hidden border border-slate-50">
      <table class="w-full text-left">
        <thead class="bg-slate-50/50 text-[9px] font-black uppercase text-slate-400 tracking-[0.2em]">
          <tr>
            <th class="px-8 py-5">Nome</th>
            <th class="px-8 py-5">Tipo</th>
            <th class="px-8 py-5 text-right">Ações</th>
          </tr>
        </thead>
        <tbody class="divide-y divide-slate-50">
          <tr v-for="cat in categories" :key="cat.id" class="hover:bg-slate-50/50 transition-colors">
            <td class="px-8 py-5 font-black text-slate-700 text-sm italic">{{ cat.name }}</td>
            <td class="px-8 py-5">
              <span :class="cat.type === 'INCOME' ? 'bg-emerald-50 text-emerald-600 border-emerald-100' : 'bg-rose-50 text-rose-600 border-rose-100'"
                class="px-3 py-1.5 rounded-full font-black uppercase tracking-tighter border text-[9px]">
                {{ cat.type === 'INCOME' ? 'Receita' : 'Despesa' }}
              </span>
            </td>
            <td class="px-8 py-5 text-right">
              <div class="flex justify-end gap-6">
                <button @click="prepareEdit(cat)" class="text-blue-600 hover:text-blue-800"><font-awesome-icon icon="fa-solid fa-pen-to-square" /></button>
                <button @click="openDeleteConfirm(cat)" class="text-rose-300 hover:text-rose-600"><font-awesome-icon icon="fa-solid fa-trash-can" /></button>
              </div>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <div v-else class="space-y-3">
      <div v-for="cat in categories" :key="cat.id" class="bg-white p-4 rounded-2xl shadow-sm border border-slate-50 flex justify-between items-center">
        <div class="flex flex-col">
          <span class="text-xs font-black text-slate-700 italic">{{ cat.name }}</span>
          <span :class="cat.type === 'INCOME' ? 'text-emerald-500' : 'text-rose-500'" class="text-[9px] font-black uppercase mt-1">
            {{ cat.type === 'INCOME' ? 'Receita' : 'Despesa' }}
          </span>
        </div>
        <div class="flex gap-4">
          <button @click="prepareEdit(cat)" class="text-blue-500 p-2"><font-awesome-icon icon="fa-solid fa-pen-to-square" /></button>
          <button @click="openDeleteConfirm(cat)" class="text-rose-300 p-2"><font-awesome-icon icon="fa-solid fa-trash-can" /></button>
        </div>
      </div>
    </div>

    <div v-if="showFormModal" class="fixed inset-0 z-[100] flex items-end md:items-center justify-center p-0 md:p-4 bg-slate-900/60 backdrop-blur-sm">
      <div class="bg-white rounded-t-[2rem] md:rounded-[2.5rem] shadow-2xl w-full max-w-md p-6 md:p-10 border border-white">
        <div class="flex justify-between items-center mb-6 md:mb-8">
          <h2 class="text-xl md:text-2xl font-black text-slate-800 italic tracking-tight">{{ isEditing ? 'Editar Categoria' : 'Nova Categoria' }}</h2>
          <button @click="showFormModal = false" class="text-slate-300 hover:text-slate-500 text-2xl"><font-awesome-icon icon="fa-solid fa-xmark" /></button>
        </div>
        
        <form @submit.prevent="saveCategory" class="space-y-4 md:space-y-6">
          <BaseInput label="Nome da Categoria" v-model="categoryForm.name" placeholder="Ex: Lazer" :error="errors.name" />
          <div>
            <label class="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2 ml-1">Tipo de Fluxo</label>
            <select v-model="categoryForm.type" class="w-full px-5 py-4 bg-slate-50 border border-slate-200 rounded-2xl focus:ring-2 focus:ring-blue-500 outline-none font-bold text-slate-700 appearance-none">
              <option value="EXPENSE">Despesa (Saída)</option>
              <option value="INCOME">Receita (Entrada)</option>
            </select>
          </div>
          <div class="flex flex-col md:flex-row gap-2 pt-4">
            <button type="button" @click="showFormModal = false" class="order-2 md:order-1 flex-1 py-4 text-slate-400 font-black uppercase text-[10px]">CANCELAR</button>
            <button type="submit" class="order-1 md:order-2 flex-1 bg-blue-600 text-white py-4 rounded-2xl font-black shadow-lg shadow-blue-200 hover:bg-blue-700 transition-all uppercase text-[10px] flex items-center justify-center gap-2">
              <font-awesome-icon icon="fa-solid fa-check" /> SALVAR
            </button>
          </div>
        </form>
      </div>
    </div>

    <ConfirmModal :show="showDeleteConfirm" title="Excluir Categoria?" :message="`Deseja realmente apagar a categoria '${categoryToDelete?.name}'?` " confirmText="Sim, Excluir" @close="showDeleteConfirm = false" @confirm="confirmDelete" />
    <ConfirmModal :show="showNoticeModal" title="Atenção" :message="apiErrorMessage" confirmText="Entendi" variant="primary" @close="showNoticeModal = false" @confirm="showNoticeModal = false" />
  </div>
</template>