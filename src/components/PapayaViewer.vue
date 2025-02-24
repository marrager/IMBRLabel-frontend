<script setup>
import { onMounted } from 'vue';

onMounted(() => {
  // Initialize any required Papaya configuration here
  console.log('Papaya viewer mounted');
});
</script>

<template>
  <div class="papaya-container">
    <iframe src="/public/Papaya-master/tests/debug_local_Tools.html"
      style="width: 100%; height: 100vh; border: none;"></iframe>
  </div>
</template>

<style scoped>
.papaya-container {
  width: 100%;
  height: 100vh;
  overflow: hidden;
  position: fixed;
  top: 0;
  left: 0;
}
</style>