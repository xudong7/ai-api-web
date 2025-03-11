<template>
  <div class="app-container">
    <router-view></router-view>
  </div>
</template>

<script setup>
import { onMounted } from 'vue'
onMounted(() => {
  // 初始化应用
})
</script>

<style scoped>
.app-container {
  width: 100%;
  min-height: 100vh;
  padding: 20px;
  box-sizing: border-box;
}

@media screen and (max-width: 768px) {
  .app-container {
    padding: 10px;
  }
}
</style>
