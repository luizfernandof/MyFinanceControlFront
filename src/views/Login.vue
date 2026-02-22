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
    // Tratamento de erro alinhado com o que vimos na sua imagem de erro inesperado
    const errorMsg = error.response?.data?.message || 'Ocorreu um erro inesperado.';
    message.value = { text: errorMsg, type: 'danger' };
  } finally {
    loading.value = false;
  }
}

function toggleMode() {
  isLogin.value = !isLogin.value;
  errors.value = { email: '', password: '' };
  message.value = { text: '', type: '' };
}
</script>

<template>
  <div class="min-h-screen bg-slate-50 flex items-center justify-center p-4 md:p-6">
    
    <div class="w-full max-w-[400px] bg-white rounded-[2.5rem] shadow-2xl shadow-slate-200/60 p-8 md:p-10 border border-white relative overflow-hidden">
      
      <div class="text-center mb-8 md:mb-10">
        <h2 class="text-3xl md:text-4xl font-black text-blue-600 tracking-tighter italic uppercase">
          {{ isLogin ? 'Login' : 'Criar Conta' }}
        </h2>
        <p class="text-slate-400 text-[12px] mt-1 font-black uppercase tracking-[0.3em] italic">
          My Finance Control
        </p>
      </div>

      <transition name="fade">
        <div v-if="message.text" 
             :class="message.type === 'success' ? 'bg-emerald-50 text-emerald-700 border-emerald-100' : 'bg-rose-50 text-rose-700 border-rose-100'"
             class="p-4 rounded-2xl mb-6 text-[11px] font-black border text-center uppercase tracking-widest leading-relaxed">
          <font-awesome-icon :icon="message.type === 'success' ? 'fa-solid fa-circle-check' : 'fa-solid fa-circle-exclamation'" class="mr-2" />
          {{ message.text }}
        </div>
      </transition>

      <form @submit.prevent="handleSubmit" novalidate class="space-y-4 md:space-y-6">
        
        <BaseInput 
          label="E-MAIL"
          v-model="email"
          type="email"
          placeholder="exemplo@email.com"
          :error="errors.email"
          inputmode="email"
          class="font-bold"
        />

        <BaseInput 
          label="SENHA"
          v-model="password"
          type="password"
          placeholder="••••••••"
          :error="errors.password"
          class="font-bold"
        />

        <button :disabled="loading" type="submit"
                class="w-full h-14 md:h-16 bg-blue-600 hover:bg-blue-700 disabled:bg-blue-300 text-white font-black rounded-2xl md:rounded-[1.5rem] shadow-xl shadow-blue-100 transition-all transform active:scale-95 flex items-center justify-center gap-3 uppercase text-xs tracking-widest mt-2">
          
          <font-awesome-icon v-if="loading" icon="fa-solid fa-spinner" spin />
          {{ isLogin ? 'Acessar Sistema' : 'Cadastrar agora' }}
        </button>
      </form>

      <div class="mt-10 text-center border-t border-slate-50 pt-8">
        <button @click="toggleMode" class="text-[10px] md:text-xs font-black text-slate-400 hover:text-blue-600 transition-colors uppercase tracking-widest">
          {{ isLogin ? 'Não tem uma conta? Registre-se' : 'Já possui conta? Faça login' }}
        </button>
      </div>
    </div>
  </div>
</template>

<style scoped>
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>