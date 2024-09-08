<template>
  <Drawer :initialPixels="pixels" ref="drawingCanvas" @updatePixels="updatePixels"/>
  <div class="panelButtons">
    <button @click="loadPixels">Load Pixels</button>
    <button @click="savePixels">Save Pixels</button>
  </div>
</template>

<script setup>
import { ref } from 'vue';
const drawingCanvas = ref(null);
const pixels = ref({});

function loadPixels() {
  const savedPixels = JSON.parse(localStorage.getItem('drawnChunks')) || {};
  console.log(savedPixels)
  pixels.value = savedPixels;
}

function savePixels() {
  localStorage.setItem('drawnChunks', JSON.stringify(pixels.value));
}

function updatePixels(newPixels) {
  pixels.value = newPixels;
}

function setPixel(x, y, color) {
  const canvas = drawingCanvas.value;
  if (canvas && canvas.setPixel) {
    canvas.setPixel(x, y, color);
  }
}
</script>

<style>
.panelButtons {
  position: fixed;
  bottom: 10px;
  left: 10px;
  display: flex;
  gap: 5px;
  z-index: 99999;
}
</style>
