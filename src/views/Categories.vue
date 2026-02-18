<script setup>
import { ref, onMounted } from 'vue';
import api from '../services/api';
import BaseInput from '../components/BaseInput.vue'; // Importando seu novo componente

// --- ESTADOS ---
const categories = ref([]);
const loading = ref(false);
const apiErrorMessage = ref('');
const errors = ref({ name: '' }); // Estado para erros do formulário

// Controles de Visibilidade
const showFormModal = ref(false);
const showDeleteModal = ref(false);
const showErrorMessage = ref(false);

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
  if (!validateForm()) return; // Validação antes do envio

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
    apiErrorMessage.value = error.response?.data?.message || "Erro inesperado ao processar.";
    showErrorMessage.value = true;
  }
}

// --- EXCLUSÃO ---
function openDeleteModal(cat) {
  categoryToDelete.value = cat;
  showDeleteModal.value = true;
}

async function confirmDelete() {
  if (categoryToDelete.value) {
    try {
      await api.delete(`/categories/${categoryToDelete.value.id}`);
      showDeleteModal.value = false;
      fetchCategories();
    } catch (error) {
      showError("Erro ao excluir. Verifique se existem transações vinculadas.");
    }
  }
}

// --- AUXILIARES ---
function openCreateModal() {
  resetForm();
  showFormModal.value = true;
}

function prepareEdit(category) {
  resetForm(); // Limpa erros antes de editar
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

function showError(msg) {
  apiErrorMessage.value = msg;
  showErrorMessage.value = true;
}

onMounted(fetchCategories);
</script>

<template>
  <div class="p-6 max-w-5xl mx-auto">
    <div class="flex justify-between items-center mb-8">
      <h2 class="text-2xl font-black text-slate-800 tracking-tight">Minhas Categorias</h2>
      <button @click="openCreateModal" 
        class="bg-blue-600 hover:bg-blue-700 text-white px-6 py-2.5 rounded-2xl font-bold shadow-lg shadow-blue-200 transition-all transform active:scale-95">
        + Nova Categoria
      </button>
    </div>

    <div class="bg-white rounded-3xl shadow-xl shadow-slate-200/50 border border-slate-100 overflow-hidden">
      <table class="w-full text-left border-collapse">
        <thead>
          <tr class="bg-slate-50/50 border-b border-slate-100 text-slate-400">
            <th class="px-6 py-4 text-[10px] font-black uppercase tracking-widest">Nome</th>
            <th class="px-6 py-4 text-[10px] font-black uppercase tracking-widest">Tipo</th>
            <th class="px-6 py-4 text-[10px] font-black uppercase tracking-widest text-right">Ações</th>
          </tr>
        </thead>
        <tbody class="divide-y divide-slate-50">
          <tr v-for="cat in categories" :key="cat.id" class="hover:bg-slate-50 transition-colors">
            <td class="px-6 py-4 font-bold text-slate-700">{{ cat.name }}</td>
            <td class="px-6 py-4 text-xs">
              <span :class="cat.type === 'INCOME' ? 'bg-emerald-50 text-emerald-600 border-emerald-100' : 'bg-rose-50 text-rose-600 border-rose-100'"
                class="px-3 py-1.5 rounded-full font-black uppercase tracking-tighter border">
                {{ cat.type === 'INCOME' ? 'Receita' : 'Despesa' }}
              </span>
            </td>
            <td class="px-6 py-4 text-right space-x-3 text-xs">
              <button @click="prepareEdit(cat)" class="text-slate-400 hover:text-blue-600 font-black uppercase transition-colors">Editar</button>
              <button @click="openDeleteModal(cat)" class="text-slate-400 hover:text-rose-600 font-black uppercase transition-colors">Excluir</button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <div v-if="showFormModal" class="fixed inset-0 z-50 flex items-center justify-center p-4 bg-slate-900/60 backdrop-blur-sm">
      <div class="bg-white rounded-[2rem] shadow-2xl w-full max-w-md p-8 border border-white/20">
        <h3 class="text-xl font-black text-slate-800 tracking-tight mb-6">
          {{ isEditing ? 'Editar Categoria' : 'Nova Categoria' }}
        </h3>
        
        <form @submit.prevent="saveCategory" novalidate class="space-y-5">
          
          <BaseInput 
            label="Nome da Categoria"
            v-model="categoryForm.name"
            placeholder="Ex: Alimentação"
            :error="errors.name"
          />

          <div>
            <label class="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2 ml-1">Tipo</label>
            <select v-model="categoryForm.type" 
              class="w-full px-5 py-3 bg-slate-50 border border-slate-200 rounded-2xl focus:ring-2 focus:ring-blue-500 outline-none bg-white transition-all shadow-sm">
              <option value="EXPENSE">Despesa</option>
              <option value="INCOME">Receita</option>
            </select>
          </div>

          <div class="flex gap-3 pt-6 border-t border-slate-50">
            <button type="button" @click="showFormModal = false" class="flex-1 py-3 text-slate-400 font-bold hover:text-slate-700 transition-colors">CANCELAR</button>
            <button type="submit" class="flex-1 bg-blue-600 text-white py-3 rounded-2xl font-black shadow-lg shadow-blue-200 hover:bg-blue-700 active:scale-95 transition-all">
              SALVAR
            </button>
          </div>
        </form>
      </div>
    </div>

    <div v-if="showDeleteModal" class="fixed inset-0 z-50 flex items-center justify-center p-4 bg-slate-900/60 backdrop-blur-sm">
      <div class="bg-white rounded-[2rem] shadow-2xl w-full max-w-xs p-8 text-center border border-white/20">
        <div class="w-16 h-16 bg-rose-50 text-rose-500 rounded-full flex items-center justify-center mx-auto mb-4">
          <svg class="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"></path></svg>
        </div>
        <h3 class="text-lg font-black text-slate-800 mb-2">Excluir?</h3>
        <p class="text-slate-400 text-xs italic mb-8">"{{ categoryToDelete?.name }}"</p>
        <div class="flex gap-2">
          <button @click="showDeleteModal = false" class="flex-1 py-3 bg-slate-100 rounded-xl font-bold text-slate-500 hover:bg-slate-200 transition-colors uppercase text-[10px]">Não</button>
          <button @click="confirmDelete" class="flex-1 py-3 bg-rose-500 text-white rounded-xl font-bold shadow-lg shadow-rose-100 hover:bg-rose-600 active:scale-95 transition-all uppercase text-[10px]">Sim</button>
        </div>
      </div>
    </div>

    <div v-if="showErrorMessage" class="fixed inset-0 z-[60] flex items-center justify-center p-4 bg-slate-900/60 backdrop-blur-sm">
      <div class="bg-white rounded-[2rem] shadow-2xl w-full max-w-xs p-8 text-center border border-white/20">
        <div class="text-rose-500 mb-4 flex justify-center">
          <svg class="w-12 h-12" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"></path></svg>
        </div>
        <h3 class="text-lg font-black text-slate-800 mb-2 tracking-tight uppercase">Aviso</h3>
        <p class="text-slate-400 text-sm mb-6">{{ apiErrorMessage }}</p>
        <button @click="showErrorMessage = false" 
          class="w-full py-3 bg-slate-800 text-white rounded-xl font-bold hover:bg-slate-900 transition-colors uppercase text-xs">Entendi</button>
      </div>
    </div>
  </div>
</template>