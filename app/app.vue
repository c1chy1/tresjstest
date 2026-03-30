<script setup lang="ts">
import { ref } from "vue";
import Cube from "./Cube.vue";
import Heart from "./Heart.vue";

const selectedShape = ref<"cube" | "heart">("cube");
</script>

<template>
  <div class="app-shell">
    <nav class="shape-nav">
      <a href="#" @click.prevent="selectedShape = 'cube'">Kostka</a>
      <a href="#" @click.prevent="selectedShape = 'heart'">Serduszko</a>
    </nav>

    <Cube v-if="selectedShape === 'cube'" />
    <Heart v-else />

    <img class="grain" src="/img/bedge-grunge.png" alt="" aria-hidden="true" />
  </div>
</template>

<style scoped>
.app-shell {
  position: relative;
  min-height: 100vh;
  background: #fdf6e3;
}

.shape-nav {
  position: fixed;
  top: 20px;
  left: 20px;
  z-index: 30;
  display: flex;
  gap: 16px;
}

.shape-nav a {
  color: #111827;
  font-weight: 600;
  text-decoration: underline;
}

.grain {
  position: fixed;
  inset: 0;
  z-index: 20;
  width: 100vw;
  height: 100vh;
  pointer-events: none;
  object-fit: cover;
  opacity: 0.3;
  mix-blend-mode: multiply;
  filter: contrast(1.08) brightness(0.96);
}

:global(html, body, #__nuxt) {
  min-height: 100%;
  margin: 0;
  background: #fdf6e3;
}
</style>
