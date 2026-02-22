<script setup>
defineProps({
  show: Boolean,
  title: String,
  message: String,
  confirmText: { type: String, default: 'Confirmar' },
  cancelText: { type: String, default: 'Cancelar' },
  variant: { type: String, default: 'danger' } // 'danger' ou 'primary'
});

defineEmits(['close', 'confirm']);
</script>

<template>
  <Transition name="fade">
    <div v-if="show" class="fixed inset-0 z-[200] flex items-center justify-center p-4 bg-slate-900/60 backdrop-blur-sm">
      <div class="bg-white rounded-[2.5rem] shadow-2xl w-full max-w-xs p-8 border border-white text-center">
        <div 
          :class="variant === 'danger' ? 'bg-rose-50 text-rose-500' : 'bg-blue-50 text-blue-500'"
          class="w-16 h-16 rounded-full flex items-center justify-center mx-auto mb-6"
        >
          <svg v-if="variant === 'danger'" class="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
          </svg>
          <svg v-else class="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
          </svg>
        </div>

        <h3 class="text-lg font-black text-slate-800 uppercase tracking-tighter mb-2">{{ title }}</h3>
        <p class="text-slate-400 text-xs italic mb-8 px-2">{{ message }}</p>

        <div class="flex gap-3">
          <button 
            @click="$emit('close')" 
            class="flex-1 py-3 text-slate-400 font-black uppercase text-[10px] tracking-widest hover:text-slate-600 transition-colors"
          >
            {{ cancelText }}
          </button>
          <button 
            @click="$emit('confirm')" 
            :class="variant === 'danger' ? 'bg-rose-500 shadow-rose-100 hover:bg-rose-600' : 'bg-blue-600 shadow-blue-100 hover:bg-blue-700'"
            class="flex-1 py-3 text-white rounded-2xl font-black shadow-lg uppercase text-[10px] tracking-widest transform active:scale-95 transition-all"
          >
            {{ confirmText }}
          </button>
        </div>
      </div>
    </div>
  </Transition>
</template>

<style scoped>
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>