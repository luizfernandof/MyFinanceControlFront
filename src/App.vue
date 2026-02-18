<script setup>
import { computed } from 'vue';
import { useRoute } from 'vue-router';
import Navbar from './components/Navbar.vue';

const route = useRoute();

// Só mostra a navbar se não estivermos na página de Login ('/')
const showNavbar = computed(() => route.path !== '/');
</script>

<template>
  <div>
    <Navbar v-if="showNavbar" />
    <router-view />
  </div>
</template>