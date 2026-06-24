<script setup>
import Gantt from './components/Gantt.vue'
</script>

<template>
  <div class="app-layout">
    <header class="header">
      <h1>Gantt Chart</h1>
    </header>
    <main>
      <Gantt />
    </main>
  </div>
</template>

<style scoped>
.app-layout {
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
}
.header {
  flex-shrink: 0;
  padding: 0 24px;
}
.header h1 {
  margin: 12px 0;
  font-size: 20px;
}
main {
  flex: 1;
  overflow: hidden;
  border-top: 1px solid #e0e0e0;
}
</style>
