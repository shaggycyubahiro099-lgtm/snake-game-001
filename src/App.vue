<template>
  <div class="container">
    <h1>Snake Game</h1>

    <h2>Score: {{ score }}</h2>

    <div class="board">
      <div
        v-for="(cell, index) in cells"
        :key="index"
        class="cell"
        :class="{
          snake: isSnake(index),
          food: isFood(index)
        }"
      ></div>
    </div>

    <div v-if="gameOver" class="game-over">
      <h2>Game Over!</h2>
      <button @click="restartGame">Restart</button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from "vue";

const width = 20;
const totalCells = width * width;

const snake = ref([45, 44, 43]);
const food = ref(120);
const score = ref(0);
const gameOver = ref(false);

let direction = 1;
let interval = null;

const cells = computed(() => Array(totalCells).fill(0));

function isSnake(index) {
  return snake.value.includes(index);
}

function isFood(index) {
  return food.value === index;
}

function generateFood() {
  let random;

  do {
    random = Math.floor(Math.random() * totalCells);
  } while (snake.value.includes(random));

  food.value = random;
}

function moveSnake() {
  if (gameOver.value) return;

  const head = snake.value[0];
  const newHead = head + direction;

  // Wall collision
  if (
    (direction === 1 && head % width === width - 1) ||
    (direction === -1 && head % width === 0) ||
    (direction === width && head >= totalCells - width) ||
    (direction === -width && head < width)
  ) {
    endGame();
    return;
  }

  // Self collision
  if (snake.value.includes(newHead)) {
    endGame();
    return;
  }

  snake.value.unshift(newHead);

  if (newHead === food.value) {
    score.value++;
    generateFood();
  } else {
    snake.value.pop();
  }
}

function endGame() {
  gameOver.value = true;
  clearInterval(interval);
}

function handleKey(event) {
  switch (event.key) {
    case "ArrowUp":
      if (direction !== width) direction = -width;
      break;

    case "ArrowDown":
      if (direction !== -width) direction = width;
      break;

    case "ArrowLeft":
      if (direction !== 1) direction = -1;
      break;

    case "ArrowRight":
      if (direction !== -1) direction = 1;
      break;
  }
}

function restartGame() {
  snake.value = [45, 44, 43];
  score.value = 0;
  direction = 1;
  gameOver.value = false;

  generateFood();

  clearInterval(interval);
  interval = setInterval(moveSnake, 200);
}

onMounted(() => {
  window.addEventListener("keydown", handleKey);

  interval = setInterval(moveSnake, 200);
});

onUnmounted(() => {
  window.removeEventListener("keydown", handleKey);
  clearInterval(interval);
});
</script>

<style>
.container {
  text-align: center;
  font-family: Arial, sans-serif;
  margin-top: 20px;
}

.board {
  width: 400px;
  height: 400px;
  margin: auto;
  display: grid;
  grid-template-columns: repeat(20, 1fr);
  border: 2px solid black;
}

.cell {
  width: 20px;
  height: 20px;
  border: 1px solid #eee;
}

.snake {
  background-color: green;
}

.food {
  background-color: red;
}

.game-over {
  margin-top: 20px;
}

button {
  padding: 10px 20px;
  cursor: pointer;
}
</style>