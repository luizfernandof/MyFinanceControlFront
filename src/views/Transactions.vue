<script setup>
import { ref, onMounted, watch } from 'vue';
import api from '../services/api';
import BaseInput from '../components/BaseInput.vue';

// --- ESTADOS ---
const now = new Date();
const selectedMonth = ref(now.getMonth() + 1);
const selectedYear = ref(now.getFullYear());

// Estados de Paginação
const currentPage = ref(0);
const totalPages = ref(0);
const pageSize = ref(10);

const transactions = ref([]);
const categories = ref([]);
const loading = ref(false);
const apiErrorMessage = ref('');
const errors = ref({});

const showFormModal = ref(false);
const showDeleteModal = ref(false);
const transactionToDelete = ref(null);
const isEditing = ref(false);

const transactionForm = ref({
    id: null,
    description: '',
    amount: 0,
    date: new Date().toISOString().split('T')[0],
    type: 'EXPENSE',
    categoryId: '',
    installments: 1
});

const months = [
  { value: 1, label: 'Janeiro' }, { value: 2, label: 'Fevereiro' },
  { value: 3, label: 'Março' }, { value: 4, label: 'Abril' },
  { value: 5, label: 'Maio' }, { value: 6, label: 'Junho' },
  { value: 7, label: 'Julho' }, { value: 8, label: 'Agosto' },
  { value: 9, label: 'Setembro' }, { value: 10, label: 'Outubro' },
  { value: 11, label: 'Novembro' }, { value: 12, label: 'Dezembro' }
];

const years = Array.from({ length: 6 }, (_, i) => now.getFullYear() - i);

// --- VALIDAÇÃO ---
function validateForm() {
    errors.value = {};
    let isValid = true;

    if (!transactionForm.value.description.trim()) {
        errors.value.description = 'A descrição é obrigatória.';
        isValid = false;
    }
    if (transactionForm.value.amount <= 0) {
        errors.value.amount = 'O valor deve ser maior que zero.';
        isValid = false;
    }
    if (!transactionForm.value.date) {
        errors.value.date = 'A data é obrigatória.';
        isValid = false;
    }
    if (!transactionForm.value.categoryId) {
        errors.value.categoryId = 'Selecione uma categoria.';
        isValid = false;
    }
    if (transactionForm.value.installments < 1) {
        errors.value.installments = 'Mínimo 1 parcela.';
        isValid = false;
    }

    return isValid;
}

// --- CARREGAMENTO ---
async function fetchData() {
    loading.value = true;
    try {
        const params = {
            month: selectedMonth.value,
            year: selectedYear.value,
            page: currentPage.value,
            size: pageSize.value,
            sort: 'date,desc' // Ordenação padrão do backend
        };

        const [resTrans, resCats] = await Promise.all([
            api.get('/transactions', { params }),
            api.get('/categories')
        ]);

        // ATENÇÃO: Agora os dados estão em .content devido à paginação do Spring
        transactions.value = resTrans.data.content;
        totalPages.value = resTrans.data.totalPages;
        categories.value = resCats.data;
    } catch (error) {
        showError(error);
    } finally {
        loading.value = false;
    }
}

// Observar mudanças de mês/ano para resetar a página e recarregar
watch([selectedMonth, selectedYear], () => {
    currentPage.value = 0;
    fetchData();
});

// --- SALVAR ---
async function saveTransaction() {
    if (!validateForm()) return;

    try {
        if (isEditing.value) {
            await api.put(`/transactions/${transactionForm.value.id}`, transactionForm.value);
        } else {
            await api.post('/transactions', transactionForm.value);
        }
        
        showFormModal.value = false;
        resetForm();
        fetchData();
    } catch (error) {
        showError(error);
    }
}

// --- EXCLUSÃO ---
function openDeleteModal(trans) {
    transactionToDelete.value = trans;
    showDeleteModal.value = true;
}

async function confirmDelete() {
    if (transactionToDelete.value) {
        try {
            await api.delete(`/transactions/${transactionToDelete.value.id}`);
            showDeleteModal.value = false;
            fetchData();
            transactionToDelete.value = null;
        } catch (error) {
            showError(error);
        }
    }
}

// --- NAVEGAÇÃO ---
function changePage(page) {
    if (page >= 0 && page < totalPages.value) {
        currentPage.value = page;
        fetchData();
    }
}

// --- AUXILIARES ---
function openCreateModal() {
    resetForm();
    showFormModal.value = true;
}

function prepareEdit(trans) {
    resetForm();
    isEditing.value = true;
    transactionForm.value = {
        id: trans.id,
        description: trans.description,
        amount: trans.amount,
        date: trans.date,
        type: trans.type,
        categoryId: categories.value.find(c => c.name === trans.categoryName)?.id || '',
        installments: 1
    };
    showFormModal.value = true;
}

function resetForm() {
    transactionForm.value = {
        id: null,
        description: '',
        amount: 0,
        date: new Date().toISOString().split('T')[0],
        type: 'EXPENSE',
        categoryId: '',
        installments: 1
    };
    isEditing.value = false;
    apiErrorMessage.value = '';
    errors.value = {};
}

function showError(error) {
    apiErrorMessage.value = error.response?.data?.message || "Erro na operação.";
}

onMounted(fetchData);
</script>

<template>
    <div class="p-6 max-w-7xl mx-auto">
        <div class="flex flex-col md:flex-row justify-between items-start md:items-center mb-8 gap-4">
            <div>
                <h2 class="text-2xl font-black text-slate-800 tracking-tight">Transações</h2>
                <p class="text-slate-400 text-sm font-medium italic">Listando: {{ months[selectedMonth-1].label }} / {{ selectedYear }}</p>
            </div>

            <div class="flex items-center gap-4">
                <div class="flex gap-2 bg-white p-2 rounded-2xl shadow-sm border border-slate-100 items-center">
                    <select v-model="selectedMonth" class="bg-transparent text-slate-600 text-xs font-bold px-2 py-1 outline-none cursor-pointer">
                        <option v-for="m in months" :key="m.value" :value="m.value">{{ m.label }}</option>
                    </select>
                    <div class="w-px bg-slate-200 h-4"></div>
                    <select v-model="selectedYear" class="bg-transparent text-slate-600 text-xs font-bold px-2 py-1 outline-none cursor-pointer">
                        <option v-for="y in years" :key="y" :value="y">{{ y }}</option>
                    </select>
                </div>

                <button @click="openCreateModal" 
                    class="bg-blue-600 hover:bg-blue-700 text-white px-6 py-3 rounded-2xl font-bold shadow-lg shadow-blue-200 transition-all transform active:scale-95 text-sm">
                    + Novo Lançamento
                </button>
            </div>
        </div>

        <div class="bg-white rounded-3xl shadow-xl shadow-slate-200/50 border border-slate-100 overflow-hidden">
            <div class="overflow-x-auto">
                <table class="w-full text-left border-collapse">
                    <thead>
                        <tr class="bg-slate-50/50 border-b border-slate-100 text-slate-400">
                            <th class="px-6 py-4 text-[10px] font-black uppercase tracking-widest">Data</th>
                            <th class="px-6 py-4 text-[10px] font-black uppercase tracking-widest">Descrição</th>
                            <th class="px-6 py-4 text-[10px] font-black uppercase tracking-widest">Categoria</th>
                            <th class="px-6 py-4 text-[10px] font-black uppercase tracking-widest text-right">Valor</th>
                            <th class="px-6 py-4 text-[10px] font-black uppercase tracking-widest text-right">Ações</th>
                        </tr>
                    </thead>
                    <tbody v-if="transactions.length > 0" class="divide-y divide-slate-50">
                        <tr v-for="t in transactions" :key="t.id" class="hover:bg-slate-50/80 transition-colors group">
                            <td class="px-6 py-4 text-xs font-medium text-slate-400">
                                {{ new Date(t.date).toLocaleDateString('pt-BR', {timeZone: 'UTC'}) }}
                            </td>
                            <td class="px-6 py-4 text-sm font-bold text-slate-700">
                                {{ t.description }}
                            </td>
                            <td class="px-6 py-4">
                                <span class="px-3 py-1 bg-white border border-slate-200 text-slate-400 rounded-full text-[10px] font-black uppercase tracking-tighter">
                                    {{ t.categoryName }}
                                </span>
                            </td>
                            <td class="px-6 py-4 text-sm font-black tracking-tight text-right" 
                                :class="t.type === 'INCOME' ? 'text-emerald-500' : 'text-rose-500'">
                                {{ t.type === 'INCOME' ? '+' : '-' }} R$ {{ (t.amount || 0).toFixed(2) }}
                            </td>
                            <td class="px-6 py-4 text-right">
                                <button @click="prepareEdit(t)" class="text-slate-300 hover:text-blue-600 font-black text-[10px] mr-4 transition-colors uppercase">Editar</button>
                                <button @click="openDeleteModal(t)" class="text-slate-300 hover:text-rose-600 font-black text-[10px] transition-colors uppercase">Excluir</button>
                            </td>
                        </tr>
                    </tbody>
                    <tbody v-else>
                        <tr>
                            <td colspan="5" class="p-20 text-center text-slate-300 italic text-sm">
                                Nenhuma transação encontrada para este período.
                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>

            <div v-if="totalPages > 1" class="px-6 py-4 bg-slate-50/50 border-t border-slate-100 flex justify-between items-center">
                <span class="text-[10px] font-bold text-slate-400 uppercase tracking-widest">
                    Página {{ currentPage + 1 }} de {{ totalPages }}
                </span>
                <div class="flex gap-2">
                    <button 
                        @click="changePage(currentPage - 1)" 
                        :disabled="currentPage === 0"
                        class="px-4 py-2 text-[10px] font-black bg-white border border-slate-200 rounded-xl disabled:opacity-30 hover:bg-slate-50 transition-colors uppercase"
                    >
                        Anterior
                    </button>
                    <button 
                        @click="changePage(currentPage + 1)" 
                        :disabled="currentPage === totalPages - 1"
                        class="px-4 py-2 text-[10px] font-black bg-white border border-slate-200 rounded-xl disabled:opacity-30 hover:bg-slate-50 transition-colors uppercase"
                    >
                        Próximo
                    </button>
                </div>
            </div>
        </div>

        <div v-if="showFormModal" class="fixed inset-0 z-50 flex items-center justify-center p-4 bg-slate-900/60 backdrop-blur-sm">
            <div class="bg-white rounded-[2rem] shadow-2xl w-full max-w-lg p-8 border border-white/20">
                <div class="flex justify-between items-center mb-6">
                    <h3 class="text-xl font-black text-slate-800 tracking-tight italic">
                        {{ isEditing ? 'Editar Lançamento' : 'Novo Lançamento' }}
                    </h3>
                    <button @click="showFormModal = false" class="text-slate-400 hover:text-slate-600 transition-colors">
                        <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path></svg>
                    </button>
                </div>

                <div v-if="apiErrorMessage" class="mb-6 p-4 bg-rose-50 text-rose-600 text-[11px] font-bold rounded-2xl border border-rose-100 uppercase tracking-wider text-center">
                    {{ apiErrorMessage }}
                </div>

                <form @submit.prevent="saveTransaction" novalidate class="space-y-5">
                    
                    <BaseInput 
                        label="Descrição"
                        v-model="transactionForm.description"
                        placeholder="Ex: Supermercado"
                        :error="errors.description"
                    />

                    <div class="grid grid-cols-2 gap-4">
                        <BaseInput 
                            label="Valor Total"
                            type="number"
                            v-model="transactionForm.amount"
                            :error="errors.amount"
                        />
                        <BaseInput 
                            label="Data"
                            type="date"
                            v-model="transactionForm.date"
                            :error="errors.date"
                        />
                    </div>

                    <div class="grid grid-cols-2 gap-4">
                        <div>
                            <label class="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2 ml-1">Tipo</label>
                            <select v-model="transactionForm.type" 
                                class="w-full px-5 py-3 bg-slate-50 border border-slate-200 rounded-2xl focus:ring-2 focus:ring-blue-500 outline-none transition-all shadow-sm">
                                <option value="EXPENSE">Despesa</option>
                                <option value="INCOME">Receita</option>
                            </select>
                        </div>
                        <div>
                            <label class="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2 ml-1">Categoria</label>
                            <select v-model="transactionForm.categoryId" 
                                :class="errors.categoryId ? 'border-rose-400 bg-rose-50' : 'border-slate-200 bg-slate-50'"
                                class="w-full px-5 py-3 border rounded-2xl focus:ring-2 focus:ring-blue-500 outline-none transition-all shadow-sm">
                                <option value="" disabled>Selecione...</option>
                                <option v-for="c in categories" :key="c.id" :value="c.id">{{ c.name }}</option>
                            </select>
                            <p v-if="errors.categoryId" class="text-rose-500 text-[11px] font-bold mt-1 ml-2">{{ errors.categoryId }}</p>
                        </div>
                    </div>

                    <div v-if="!isEditing && transactionForm.type === 'EXPENSE'">
                        <BaseInput 
                            label="Nº de Parcelas"
                            type="number"
                            v-model="transactionForm.installments"
                            placeholder="Ex: 12"
                            :error="errors.installments"
                        />
                    </div>

                    <div class="flex gap-3 pt-6 border-t border-slate-50 mt-4">
                        <button type="button" @click="showFormModal = false" 
                            class="flex-1 py-3 text-slate-400 font-bold hover:text-slate-600 transition-colors uppercase text-xs">
                            CANCELAR
                        </button>
                        <button type="submit" 
                            class="flex-1 bg-blue-600 text-white py-3 rounded-2xl font-black shadow-lg shadow-blue-200 hover:bg-blue-700 transition-all transform active:scale-95 uppercase text-xs">
                            SALVAR
                        </button>
                    </div>
                </form>
            </div>
        </div>

        <div v-if="showDeleteModal" class="fixed inset-0 z-50 flex items-center justify-center p-4 bg-slate-900/60 backdrop-blur-sm text-center">
            <div class="bg-white rounded-[2rem] shadow-2xl w-full max-w-xs p-8 border border-white/20">
                <div class="w-16 h-16 bg-rose-50 text-rose-500 rounded-full flex items-center justify-center mx-auto mb-4">
                    <svg class="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"></path></svg>
                </div>
                <h3 class="text-lg font-black text-slate-800 mb-2 uppercase tracking-tighter">Apagar Registro?</h3>
                <p class="text-slate-400 text-xs italic mb-8">"{{ transactionToDelete?.description }}"</p>
                <div class="flex gap-3">
                    <button @click="showDeleteModal = false" class="flex-1 py-3 text-slate-400 font-bold hover:text-slate-600 transition-colors uppercase text-[10px]">Não</button>
                    <button @click="confirmDelete" class="flex-1 py-3 bg-rose-500 text-white rounded-xl font-black shadow-lg shadow-rose-100 hover:bg-rose-600 transition-all transform active:scale-95 uppercase text-[10px]">Sim</button>
                </div>
            </div>
        </div>
    </div>
</template>