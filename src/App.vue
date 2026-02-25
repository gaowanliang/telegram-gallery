<script setup>
import { ref, onMounted, watch, provide } from 'vue';
import Login from './components/Login.vue';
import Gallery from './components/Gallery.vue';

const token = ref(localStorage.getItem('gallery_token'));
const theme = ref(localStorage.getItem('gallery_theme') || 'light');

function onLogin() {
  token.value = localStorage.getItem('gallery_token');
}

function toggleTheme() {
  theme.value = theme.value === 'light' ? 'dark' : 'light';
}

provide('theme', theme);
provide('toggleTheme', toggleTheme);

onMounted(() => {
  document.documentElement.setAttribute('data-theme', theme.value);
});

watch(theme, (newTheme) => {
  document.documentElement.setAttribute('data-theme', newTheme);
  localStorage.setItem('gallery_theme', newTheme);
});
</script>

<template>
  <div id="app">
    <transition name="fade" mode="out-in">
      <Gallery v-if="token" key="gallery" />
      <Login v-else key="login" @login="onLogin" />
    </transition>
  </div>
</template>

<style scoped>
#app {
  min-height: 100vh;
  position: relative;
}

.theme-toggle {
  position: fixed;
  top: 1rem;
  right: 1rem;
  z-index: 1000;
  width: 2.5rem;
  height: 2.5rem;
  border-radius: 50%;
  background-color: var(--bg-secondary);
  border: 1px solid var(--border-color);
  box-shadow: var(--shadow);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.25rem;
  padding: 0;
  transition: all 0.2s;
}

.theme-toggle:hover {
  transform: scale(1.1);
  box-shadow: var(--shadow-lg);
}

.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
</style>
