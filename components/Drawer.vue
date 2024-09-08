<template>
  <div>
    <div class="color-palette">
      <button v-for="color in colors" :key="color" :style="{ backgroundColor: color }" @click="setColor(color)"></button>
    </div>
    <canvas 
      ref="canvas" 
      @mousedown.left="startDrawing" 
      @mousemove="handleMouseMove" 
      @mouseup="stopDrawing" 
      @mouseleave="stopDrawing" 
      @wheel="zoomCanvas" 
      @mousedown.middle="startPanning"
    ></canvas>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, defineProps, defineEmits } from 'vue';

const props = defineProps({
  initialPixels: Object
});

const emit = defineEmits(['updatePixels']);

const canvas = ref(null);
const isDrawing = ref(false);
const isPanning = ref(false);
const ctx = ref(null);
const imgCtx = ref(null);

const canvasSize = reactive({ width: window.innerWidth, height: window.innerHeight });
const squareSize = 512;
const cellSize = 4;
const chunkSize = 64;
const squareX = Math.round((canvasSize.width - squareSize) / 2);
const squareY = Math.round((canvasSize.height - squareSize) / 2);

const zoomFactor = 1.05;
const minZoom = 1;
const maxZoom = 6;

const state = reactive({
  scale: 1,
  offsetX: 0,
  offsetY: 0,
  color: '#000'
});

let lastMouseX = null;
let lastMouseY = null;
const drawnChunks = ref({ ...props.initialPixels });

const colors = ['#000', '#f00', '#0f0', '#00f', '#ff0', '#0ff', '#f0f', '#fff'];

onMounted(() => {
  const el = canvas.value;
  el.width = canvasSize.width;
  el.height = canvasSize.height;
  ctx.value = el.getContext('2d');
  ctx.value.imageSmoothingEnabled = false;

  const imgCanvas = document.createElement('canvas');
  imgCanvas.width = canvasSize.width;
  imgCanvas.height = canvasSize.height;
  imgCtx.value = imgCanvas.getContext('2d');

  drawSavedChunks();
  drawCanvas();
});

function drawSavedChunks() {
  const c = imgCtx.value;
  for (const [key, pixels] of Object.entries(drawnChunks.value)) {
    pixels.forEach(({ x, y, color }) => {
      c.fillStyle = color;
      c.fillRect(x, y, cellSize, cellSize);
    });
  }
}

function drawCanvas() {
  const c = ctx.value;

  c.clearRect(0, 0, canvasSize.width, canvasSize.height);
  c.save();
  c.fillStyle = '#333';
  c.fillRect(0, 0, canvasSize.width, canvasSize.height);
  c.translate(state.offsetX, state.offsetY);
  c.scale(state.scale, state.scale);
  c.fillStyle = 'white';
  c.fillRect(squareX + 0.1, squareY + 0.1, squareSize - 0.2, squareSize - 0.2);
  c.drawImage(imgCtx.value.canvas, 0, 0);
  drawGrid();
  c.restore();
}

function drawGrid() {
  const c = ctx.value;
  c.strokeStyle = 'rgba(0,0,0,0.3)';
  c.lineWidth = 0.2;
  c.beginPath();
  for (let x = squareX; x <= squareX + squareSize; x += cellSize) {
    c.moveTo(x, squareY);
    c.lineTo(x, squareY + squareSize);
  }
  for (let y = squareY; y <= squareY + squareSize; y += cellSize) {
    c.moveTo(squareX, y);
    c.lineTo(squareX + squareSize, y);
  }
  c.stroke();
}

function startDrawing(event) {
  if (event.width == 2) return;
  if (isInsideSquare(event)) {
    isDrawing.value = true;
    draw(event);
  }
}

function stopDrawing() {
  isDrawing.value = false;
}

function handleMouseMove(event) {
  if (isDrawing.value) {
    draw(event);
  }
  if (isPanning.value) {
    panCanvas(event);
  }
}

function draw(event) {
  if (event.width == 2) return;
  if (!isDrawing.value) return;

  const c = imgCtx.value;
  const rect = canvas.value.getBoundingClientRect();
  const x = (event.clientX - rect.left - state.offsetX) / state.scale;
  const y = (event.clientY - rect.top - state.offsetY) / state.scale;

  const cellX = Math.floor((x - squareX) / cellSize) * cellSize + squareX;
  const cellY = Math.floor((y - squareY) / cellSize) * cellSize + squareY;

  const drawX = Math.round(cellX);
  const drawY = Math.round(cellY);

  if (isInsideSquare({ clientX: drawX * state.scale + state.offsetX, clientY: drawY * state.scale + state.offsetY })) {
    const chunkKey = getChunkKey(drawX, drawY);
    if (!drawnChunks.value[chunkKey]) {
      drawnChunks.value[chunkKey] = [];
    }
    drawnChunks.value[chunkKey].push({ x: drawX, y: drawY, color: state.color });
    emit('updatePixels', drawnChunks.value);

    c.fillStyle = state.color;
    c.fillRect(drawX, drawY, cellSize, cellSize);
    drawCanvas();
  }
}

function getChunkKey(x, y) {
  const chunkX = Math.floor((x - squareX) / chunkSize);
  const chunkY = Math.floor((y - squareY) / chunkSize);
  return `${chunkX}-${chunkY}`;
}

function isInsideSquare(event) {
  const rect = canvas.value.getBoundingClientRect();
  const x = (event.clientX - rect.left - state.offsetX) / state.scale;
  const y = (event.clientY - rect.top - state.offsetY) / state.scale;
  return x >= squareX && x <= squareX + squareSize && y >= squareY && y <= squareY + squareSize;
}

function zoomCanvas(event) {
  event.preventDefault();
  const c = ctx.value;
  const rect = canvas.value.getBoundingClientRect();
  const mouseX = (event.clientX - rect.left - state.offsetX) / state.scale;
  const mouseY = (event.clientY - rect.top - state.offsetY) / state.scale;
  
  const zoom = event.deltaY < 0 ? zoomFactor : 1 / zoomFactor;
  const newScale = Math.min(Math.max(state.scale * zoom, minZoom), maxZoom);
  
  state.offsetX += (state.scale - newScale) * mouseX;
  state.offsetY += (state.scale - newScale) * mouseY;
  state.scale = newScale;
  
  drawCanvas();
}

function startPanning(event) {
  isPanning.value = true;
  lastMouseX = event.clientX;
  lastMouseY = event.clientY;
  document.addEventListener('mousemove', panCanvas);
  document.addEventListener('mouseup', stopPanning);
}

function panCanvas(event) {
  if (!isPanning.value) return;

  const dx = (event.clientX - lastMouseX);
  const dy = (event.clientY - lastMouseY);

  state.offsetX += dx;
  state.offsetY += dy;

  lastMouseX = event.clientX;
  lastMouseY = event.clientY;

  drawCanvas();
}

function stopPanning() {
  isPanning.value = false;
  document.removeEventListener('mousemove', panCanvas);
  document.removeEventListener('mouseup', stopPanning);
}

function setColor(color) {
  state.color = color;
}

function setPixel(x, y, color) {
  const chunkKey = getChunkKey(x, y);
  if (!drawnChunks.value[chunkKey]) {
    drawnChunks.value[chunkKey] = [];
  }
  drawnChunks.value[chunkKey].push({ x, y, color: colors[color] });
  emit('updatePixels', drawnChunks.value);
  console.log(x,y,colors[color])
  const c = imgCtx.value;
  c.fillStyle = colors[color];
  c.fillRect(x, y, cellSize, cellSize);
  drawCanvas();
}


watch(() => props.initialPixels, (newPixels) => {
  drawnChunks.value = { ...newPixels }; 
  drawSavedChunks();
  drawCanvas();
});

defineExpose({setPixel});
</script>

<style>
canvas {
  display: block;
  position: absolute;
  top: 0;
  left: 0;
  cursor: grab;
}
canvas:active {
  cursor: grabbing;
}
.color-palette {
  position: absolute;
  top: 10px;
  left: 10px;
  display: flex;
  gap: 5px;
  z-index: 999;
}
.color-palette button {
  width: 30px;
  height: 30px;
  border: none;
  cursor: pointer;
}
</style>
