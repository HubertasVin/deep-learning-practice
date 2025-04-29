<script setup lang="ts">
import ImageSegmentation from './components/ImageSegmentation.vue'
</script>

<template>
  <div class="app">
    <h1>Image Segmentation</h1>
    <ImageSegmentation />
  </div>
</template>

<style>
.app {
  max-width: 1280px;
  margin: 0 auto;
  padding: 2rem;
}

h1 {
  text-align: center;
  margin-bottom: 2rem;
}
</style>