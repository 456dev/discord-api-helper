<script setup lang="ts">
import ServerViewer from './components/ServerViewer.vue'
</script>

<template>
  <main>
    <ServerViewer />
  </main>
</template>

<style>
:root {
  --primary-color: #5865f2;
  --secondary-color: #23272a;
  --accent-color: #7289da;
  --text-color: #ffffff;
  --background-color: #36393f;
}

body {
  margin: 0;
  font-family: 'Arial', sans-serif;
  background-color: var(--background-color);
  color: var(--text-color);
}

main {
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>
