<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from "vue";

const canvasRef = ref(null);
const score = ref(0);
const best = ref(Number(localStorage.getItem("codex-snake-best") || 0));
const paused = ref(false);
const ended = ref(false);
const started = ref(false);
const message = ref("游戏结束");

const gridSize = 24;
const canvasSize = 480;
const cellSize = canvasSize / gridSize;
const tickMs = 110;
const storageKey = "codex-snake-best";

let ctx;
let timerId;
let snake = [];
let food = { x: 0, y: 0 };
let direction = { x: 1, y: 0 };
let queuedDirection = { x: 1, y: 0 };
let touchStart = null;

const finalScore = computed(() => score.value);

function resetGame() {
  const middle = Math.floor(gridSize / 2);
  snake = [
    { x: middle, y: middle },
    { x: middle - 1, y: middle },
    { x: middle - 2, y: middle }
  ];
  direction = { x: 1, y: 0 };
  queuedDirection = { x: 1, y: 0 };
  score.value = 0;
  paused.value = false;
  ended.value = false;
  started.value = false;
  message.value = "游戏结束";
  placeFood();
  draw();
  restartTimer();
}

function restartTimer() {
  clearInterval(timerId);
  timerId = window.setInterval(update, tickMs);
}

function placeFood() {
  const occupied = new Set(snake.map((part) => `${part.x},${part.y}`));
  const emptyCells = [];

  for (let y = 0; y < gridSize; y += 1) {
    for (let x = 0; x < gridSize; x += 1) {
      if (!occupied.has(`${x},${y}`)) {
        emptyCells.push({ x, y });
      }
    }
  }

  food = emptyCells[Math.floor(Math.random() * emptyCells.length)];
}

function update() {
  if (paused.value || ended.value || !started.value) {
    return;
  }

  direction = queuedDirection;
  const head = snake[0];
  const nextHead = {
    x: head.x + direction.x,
    y: head.y + direction.y
  };

  const hitsWall =
    nextHead.x < 0 ||
    nextHead.x >= gridSize ||
    nextHead.y < 0 ||
    nextHead.y >= gridSize;

  const tail = snake[snake.length - 1];
  const eatsFood = nextHead.x === food.x && nextHead.y === food.y;
  const hitsSelf = snake.some((part) => {
    const isTailMovingAway = !eatsFood && part === tail;
    return !isTailMovingAway && part.x === nextHead.x && part.y === nextHead.y;
  });

  if (hitsWall || hitsSelf) {
    finishGame("游戏结束");
    return;
  }

  snake.unshift(nextHead);

  if (eatsFood) {
    score.value += 1;
    best.value = Math.max(best.value, score.value);
    localStorage.setItem(storageKey, String(best.value));

    if (snake.length === gridSize * gridSize) {
      finishGame("你赢了！");
      return;
    }

    placeFood();
  } else {
    snake.pop();
  }

  draw();
}

function draw() {
  if (!ctx) {
    return;
  }

  ctx.clearRect(0, 0, canvasSize, canvasSize);
  drawGrid();
  drawFood();
  drawSnake();

  if (!started.value && !ended.value) {
    drawStartHint();
  }

  if (paused.value) {
    drawPause();
  }
}

function drawGrid() {
  ctx.fillStyle = "#101722";
  ctx.fillRect(0, 0, canvasSize, canvasSize);
  ctx.strokeStyle = "rgba(255, 255, 255, 0.055)";
  ctx.lineWidth = 1;

  for (let i = 1; i < gridSize; i += 1) {
    const offset = i * cellSize;
    ctx.beginPath();
    ctx.moveTo(offset, 0);
    ctx.lineTo(offset, canvasSize);
    ctx.moveTo(0, offset);
    ctx.lineTo(canvasSize, offset);
    ctx.stroke();
  }
}

function drawFood() {
  const centerX = food.x * cellSize + cellSize / 2;
  const centerY = food.y * cellSize + cellSize / 2;

  ctx.save();
  ctx.shadowColor = "#ff5c7a";
  ctx.shadowBlur = 18;
  ctx.fillStyle = "#ff5c7a";
  ctx.beginPath();
  ctx.arc(centerX, centerY, cellSize * 0.34, 0, Math.PI * 2);
  ctx.fill();
  ctx.restore();
}

function drawSnake() {
  snake.forEach((part, index) => {
    const isHead = index === 0;
    const inset = isHead ? 2 : 3;
    const x = part.x * cellSize + inset;
    const y = part.y * cellSize + inset;
    const size = cellSize - inset * 2;

    ctx.save();
    ctx.fillStyle = isHead ? "#7af0a8" : "#50d98d";
    ctx.shadowColor = isHead ? "#7af0a8" : "#50d98d";
    ctx.shadowBlur = isHead ? 14 : 6;
    roundedRect(x, y, size, size, 5);
    ctx.fill();
    ctx.restore();

    if (isHead) {
      drawEyes(part);
    }
  });
}

function drawEyes(head) {
  const baseX = head.x * cellSize;
  const baseY = head.y * cellSize;

  eyePositions(baseX, baseY).forEach(([x, y]) => {
    ctx.fillStyle = "#101722";
    ctx.beginPath();
    ctx.arc(x, y, 2.6, 0, Math.PI * 2);
    ctx.fill();
  });
}

function eyePositions(baseX, baseY) {
  if (direction.x === 1) {
    return [[baseX + 15, baseY + 8], [baseX + 15, baseY + 16]];
  }
  if (direction.x === -1) {
    return [[baseX + 9, baseY + 8], [baseX + 9, baseY + 16]];
  }
  if (direction.y === -1) {
    return [[baseX + 8, baseY + 9], [baseX + 16, baseY + 9]];
  }
  return [[baseX + 8, baseY + 15], [baseX + 16, baseY + 15]];
}

function drawPause() {
  ctx.fillStyle = "rgba(9, 12, 17, 0.6)";
  ctx.fillRect(0, 0, canvasSize, canvasSize);
  ctx.fillStyle = "#f4f7fb";
  ctx.font = "700 42px 'Segoe UI', 'Microsoft YaHei', Arial, sans-serif";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.fillText("已暂停", canvasSize / 2, canvasSize / 2);
}

function drawStartHint() {
  ctx.fillStyle = "rgba(9, 12, 17, 0.54)";
  ctx.fillRect(0, 0, canvasSize, canvasSize);
  ctx.fillStyle = "#f4f7fb";
  ctx.font = "700 32px 'Segoe UI', 'Microsoft YaHei', Arial, sans-serif";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.fillText("按方向键开始", canvasSize / 2, canvasSize / 2 - 18);
  ctx.fillStyle = "#a7b0bd";
  ctx.font = "500 18px 'Segoe UI', 'Microsoft YaHei', Arial, sans-serif";
  ctx.fillText("也可以点击棋盘直接出发", canvasSize / 2, canvasSize / 2 + 24);
}

function roundedRect(x, y, width, height, radius) {
  ctx.beginPath();
  ctx.moveTo(x + radius, y);
  ctx.arcTo(x + width, y, x + width, y + height, radius);
  ctx.arcTo(x + width, y + height, x, y + height, radius);
  ctx.arcTo(x, y + height, x, y, radius);
  ctx.arcTo(x, y, x + width, y, radius);
  ctx.closePath();
}

function finishGame(nextMessage) {
  ended.value = true;
  clearInterval(timerId);
  message.value = nextMessage;
}

function queueDirection(next) {
  if (paused.value || ended.value) {
    return;
  }

  const isOpposite =
    next.x + direction.x === 0 &&
    next.y + direction.y === 0;

  if (!isOpposite) {
    queuedDirection = next;
    started.value = true;
  }
}

function handleKeydown(event) {
  const key = event.key.toLowerCase();
  const controls = ["arrowup", "arrowdown", "arrowleft", "arrowright", " ", "w", "a", "s", "d"];

  if (controls.includes(key)) {
    event.preventDefault();
  }

  if (key === " ") {
    if (!ended.value && started.value) {
      paused.value = !paused.value;
      draw();
    }
    return;
  }

  const directions = {
    arrowup: { x: 0, y: -1 },
    w: { x: 0, y: -1 },
    arrowdown: { x: 0, y: 1 },
    s: { x: 0, y: 1 },
    arrowleft: { x: -1, y: 0 },
    a: { x: -1, y: 0 },
    arrowright: { x: 1, y: 0 },
    d: { x: 1, y: 0 }
  };

  if (directions[key]) {
    queueDirection(directions[key]);
  }
}

function handleTouchStart(event) {
  const touch = event.touches[0];
  touchStart = { x: touch.clientX, y: touch.clientY };
}

function handleTouchEnd(event) {
  if (!touchStart) {
    return;
  }

  const touch = event.changedTouches[0];
  const dx = touch.clientX - touchStart.x;
  const dy = touch.clientY - touchStart.y;
  const minSwipe = 24;

  touchStart = null;

  if (Math.max(Math.abs(dx), Math.abs(dy)) < minSwipe) {
    return;
  }

  if (Math.abs(dx) > Math.abs(dy)) {
    queueDirection(dx > 0 ? { x: 1, y: 0 } : { x: -1, y: 0 });
  } else {
    queueDirection(dy > 0 ? { x: 0, y: 1 } : { x: 0, y: -1 });
  }
}

function startFromCanvas() {
  if (!ended.value && !started.value) {
    started.value = true;
    draw();
  }
}

onMounted(async () => {
  await nextTick();
  ctx = canvasRef.value.getContext("2d");
  document.addEventListener("keydown", handleKeydown);
  resetGame();
});

onBeforeUnmount(() => {
  clearInterval(timerId);
  document.removeEventListener("keydown", handleKeydown);
});
</script>

<template>
  <section class="snake-game" aria-label="贪吃蛇游戏">
    <div class="snake-toolbar">
      <div class="stats" aria-live="polite">
        <div class="stat">得分 <b>{{ score }}</b></div>
        <div class="stat">最高 <b>{{ best }}</b></div>
      </div>
      <button class="primary-button" type="button" @click="resetGame">重新开始</button>
    </div>

    <div class="board-wrap">
      <canvas
        ref="canvasRef"
        width="480"
        height="480"
        aria-label="贪吃蛇棋盘"
        @click="startFromCanvas"
        @touchstart.passive="handleTouchStart"
        @touchend.passive="handleTouchEnd"
      ></canvas>

      <div class="overlay" :class="{ show: ended }">
        <div>
          <h2>{{ message }}</h2>
          <p>本局得分：<span>{{ finalScore }}</span></p>
          <button class="primary-button" type="button" @click="resetGame">再来一局</button>
        </div>
      </div>
    </div>

    <p class="tip">
      <kbd>↑</kbd> <kbd>↓</kbd> <kbd>←</kbd> <kbd>→</kbd>
      或 <kbd>W</kbd> <kbd>A</kbd> <kbd>S</kbd> <kbd>D</kbd> 控制方向，
      <kbd>Space</kbd> 暂停/继续，手机可滑动控制。
    </p>
  </section>
</template>
