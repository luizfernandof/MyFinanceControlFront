<script setup>
import { ref } from 'vue';
import { useRouter } from 'vue-router';
import api from '../services/api';
import BaseInput from '../components/BaseInput.vue';

const router = useRouter();

// Estados
const isLogin = ref(true);
const email = ref('');
const password = ref('');
const loading = ref(false);
const message = ref({ text: '', type: '' });

// Estado para validações manuais
const errors = ref({ email: '', password: '' });

function validateForm() {
  let isValid = true;
  errors.value = { email: '', password: '' };

  if (!email.value) {
    errors.value.email = 'O e-mail é obrigatório.';
    isValid = false;
  } else if (!/\S+@\S+\.\S+/.test(email.value)) {
    errors.value.email = 'Insira um e-mail válido.';
    isValid = false;
  }

  if (!password.value) {
    errors.value.password = 'A senha é obrigatória.';
    isValid = false;
  } else if (password.value.length < 6) {
    errors.value.password = 'A senha deve ter pelo menos 6 caracteres.';
    isValid = false;
  }

  return isValid;
}

async function handleSubmit() {
  if (!validateForm()) return;

  message.value = { text: '', type: '' };
  loading.value = true;

  const endpoint = isLogin.value ? '/auth/login' : '/auth/register';

  try {
    const response = await api.post(endpoint, {
      email: email.value,
      password: password.value
    });

    if (isLogin.value) {
      localStorage.setItem('accessToken', response.data.accessToken);
      localStorage.setItem('refreshToken', response.data.refreshToken);
      router.push('/dashboard');
    } else {
      message.value = { text: 'Conta criada com sucesso! Faça o login.', type: 'success' };
      isLogin.value = true;
      password.value = '';
    }
  } catch (error) {
    const errorMsg = error.response?.data?.message || 'Ocorreu um erro inesperado.';
    message.value = { text: errorMsg, type: 'danger' };
  } finally {
    loading.value = false;
  }
}

// Limpa erros ao alternar entre login e cadastro
function toggleMode() {
  isLogin.value = !isLogin.value;
  errors.value = { email: '', password: '' };
  message.value = { text: '', type: '' };
}
</script>

<template>
  <div class="min-h-screen bg-slate-50 flex items-center justify-center p-4">
    
    <div class="max-w-md w-full bg-white rounded-3xl shadow-xl shadow-slate-200/60 p-8 border border-slate-100">
      
      <div class="text-center mb-8">
        <h2 class="text-3xl font-black text-blue-600 tracking-tight">
          {{ isLogin ? 'Login' : 'Criar Conta' }}
        </h2>
        <p class="text-slate-400 text-sm mt-2 font-medium tracking-wide italic">My Finance Control</p>
      </div>

      <div v-if="message.text" 
           :class="message.type === 'success' ? 'bg-emerald-50 text-emerald-700 border-emerald-100' : 'bg-rose-50 text-rose-700 border-rose-100'"
           class="p-4 rounded-2xl mb-6 text-sm font-bold border text-center transition-all duration-300">
        {{ message.text }}
      </div>

      <form @submit.prevent="handleSubmit" novalidate class="space-y-5">
        
        <BaseInput 
          label="E-mail"
          v-model="email"
          type="email"
          placeholder="exemplo@email.com"
          :error="errors.email"
        />

        <BaseInput 
          label="Senha"
          v-model="password"
          type="password"
          placeholder="••••••••"
          :error="errors.password"
        />

        <button :disabled="loading" type="submit"
                class="w-full bg-blue-600 hover:bg-blue-700 disabled:bg-blue-300 text-white font-bold py-4 rounded-2xl shadow-lg shadow-blue-200 transition-all transform active:scale-[0.98] flex items-center justify-center mt-2">
          
          <svg v-if="loading" class="animate-spin h-5 w-5 mr-3 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
          </svg>

          {{ isLogin ? 'Acessar Sistema' : 'Cadastrar agora' }}
        </button>
      </form>

      <div class="mt-8 text-center border-t border-slate-50 pt-6">
        <button @click="toggleMode" class="text-sm font-bold text-slate-400 hover:text-blue-600 transition-colors">
          {{ isLogin ? 'Não tem uma conta? Registre-se' : 'Já possui conta? Faça login' }}
        </button>
      </div>
    </div>
  </div>
</template>