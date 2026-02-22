<script setup>
import { ref, onMounted } from 'vue';
import api from '../services/api';
import BaseInput from '../components/BaseInput.vue';
import ConfirmModal from '../components/ConfirmModal.vue';

// --- ESTADOS ---
const categories = ref([]);
const loading = ref(false);
const apiErrorMessage = ref('');
const errors = ref({ name: '' });

// Controles de Visibilidade
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
      showError("Não foi possível excluir. Verifique se existem transações vinculadas a esta categoria.");
    }
  }
}

// --- AUXILIARES ---
function openCreateModal() {
  resetForm();
  showFormModal.value = true;
}

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

function showError(msg) {
  apiErrorMessage.value = msg;
  showNoticeModal.value = true;
}

onMounted(fetchCategories);
</script>

<template>
  <div class="p-6 max-w-5xl mx-auto min-h-screen">
    <div class="flex justify-between items-center mb-10">
      <div>
        <h2 class="text-3xl font-black text-slate-800 italic uppercase tracking-tighter">Minhas Categorias</h2>
        <p class="text-slate-400 text-[10px] font-black uppercase tracking-[0.3em]">Gerencie seus grupos de custo</p>
      </div>

      <button @click="openCreateModal" 
        class="bg-blue-600 text-white px-8 py-4 rounded-[1.5rem] font-black shadow-xl shadow-blue-100 hover:scale-105 transition-all text-xs uppercase tracking-widest flex items-center gap-2">
        <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="3" d="M12 4v16m8-8H4"/></svg>
        Nova Categoria
      </button>
    </div>

    <div class="bg-white rounded-[2.5rem] shadow-xl overflow-hidden border border-slate-50">
      <table class="w-full text-left">
        <thead class="bg-slate-50/50 text-[9px] font-black uppercase text-slate-400 tracking-[0.2em]">
          <tr>
            <th class="px-8 py-5">Nome</th>
            <th class="px-8 py-5">Tipo</th>
            <th class="px-8 py-5 text-right">Ações</th>
          </tr>
        </thead>
        <tbody class="divide-y divide-slate-50">
          <tr v-for="cat in categories" :key="cat.id" class="hover:bg-slate-50/50 transition-colors group">
            <td class="px-8 py-5 font-black text-slate-700 text-sm italic">{{ cat.name }}</td>
            <td class="px-8 py-5 text-xs">
              <span :class="cat.type === 'INCOME' ? 'bg-emerald-50 text-emerald-600 border-emerald-100' : 'bg-rose-50 text-rose-600 border-rose-100'"
                class="px-3 py-1.5 rounded-full font-black uppercase tracking-tighter border text-[10px]">
                {{ cat.type === 'INCOME' ? 'Receita' : 'Despesa' }}
              </span>
            </td>
            <td class="px-8 py-5 text-right">
              <div class="flex justify-end gap-6">
                <button @click="prepareEdit(cat)" class="flex items-center gap-1 text-[10px] font-black text-blue-600 hover:text-blue-800 uppercase tracking-widest transition-colors">
                  <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15.232 5.232l3.536 3.536m-2.036-5.036a2.5 2.5 0 113.536 3.536L6.5 21.036H3v-3.572L16.732 3.732z"/></svg>
                  Editar
                </button>
                <button @click="openDeleteConfirm(cat)" class="flex items-center gap-1 text-[10px] font-black text-rose-300 hover:text-rose-600 uppercase tracking-widest transition-colors">
                  <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"/></svg>
                  Excluir
                </button>
              </div>
            </td>
          </tr>
          <tr v-if="categories.length === 0 && !loading">
            <td colspan="3" class="px-8 py-20 text-center text-slate-300 italic text-sm">
              Nenhuma categoria encontrada.
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <div v-if="showFormModal" class="fixed inset-0 z-[100] flex items-center justify-center p-4 bg-slate-900/60 backdrop-blur-sm">
      <div class="bg-white rounded-[2.5rem] shadow-2xl w-full max-w-md p-10 border border-white">
        <div class="flex justify-between items-center mb-8">
          <h2 class="text-2xl font-black text-slate-800 italic tracking-tight">
            {{ isEditing ? 'Editar Categoria' : 'Nova Categoria' }}
          </h2>
          <button @click="showFormModal = false" class="text-slate-300 hover:text-slate-500 text-3xl transition-colors">&times;</button>
        </div>
        
        <form @submit.prevent="saveCategory" novalidate class="space-y-6">
          <BaseInput 
            label="Nome da Categoria"
            v-model="categoryForm.name"
            placeholder="Ex: Lazer"
            :error="errors.name"
          />

          <div>
            <label class="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2 ml-1">Tipo de Fluxo</label>
            <select v-model="categoryForm.type" 
              class="w-full px-5 py-4 bg-slate-50 border border-slate-200 rounded-2xl focus:ring-2 focus:ring-blue-500 outline-none transition-all font-bold text-slate-700">
              <option value="EXPENSE">Despesa (Saída)</option>
              <option value="INCOME">Receita (Entrada)</option>
            </select>
          </div>

          <div class="flex gap-4 pt-6">
            <button type="button" @click="showFormModal = false" class="flex-1 py-4 text-slate-400 font-black uppercase text-[10px] tracking-widest">CANCELAR</button>
            <button type="submit" class="flex-1 bg-blue-600 text-white py-4 rounded-2xl font-black shadow-lg shadow-blue-200 hover:bg-blue-700 transition-all transform active:scale-95 uppercase text-[10px] tracking-widest flex items-center justify-center gap-2">
              <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7"/></svg>
              SALVAR
            </button>
          </div>
        </form>
      </div>
    </div>

    <ConfirmModal 
      :show="showDeleteConfirm"
      title="Excluir Categoria?"
      :message="`Deseja realmente apagar a categoria '${categoryToDelete?.name}'?` "
      confirmText="Sim, Excluir"
      @close="showDeleteConfirm = false"
      @confirm="confirmDelete"
    />

    <ConfirmModal 
      :show="showNoticeModal"
      title="Atenção"
      :message="apiErrorMessage"
      confirmText="Entendi"
      variant="primary"
      @close="showNoticeModal = false"
      @confirm="showNoticeModal = false"
    />
  </div>
</template>