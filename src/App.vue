<script setup lang="ts">
import { RouterView } from 'vue-router'
</script>

<template>
  <RouterView />
</template>

<style>
#app {
  max-width: 1280px;
  margin: 0 auto;
  padding: 2rem;
  font-weight: normal;
}

/* 移动端适配 */
@media (max-width: 768px) {
  #app {
    max-width: 100%;
    margin: 0;
    padding: 0;
    height: calc(var(--vh, 1vh) * 100);
    overflow: hidden;
  }
}
</style>
