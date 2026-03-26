<template>
  <div class="game-container"> <!-- контейнер игры -->
    <div class="header"> <!-- заголовок -->
      <h1>Лабиринт Закрасчик</h1> <!-- текст заголовка -->
    </div>

    <div class="stats"> <!-- статистика игрока -->
      <div class="stat-item"> <!-- уровень -->
        <span class="label">Уровень</span>
        <span class="value">{{ level + 1 }}</span>
      </div>
      
      <div class="stat-item"> <!-- время -->
        <span class="label">⏱ Время</span>
        <span class="value">{{ time }} сек</span>
      </div>
      
      <div class="stat-item"> <!-- закрашено -->
        <span class="label">Закрашено</span>
        <span class="value">{{ paintedCount }} / {{ totalFloors }}</span>
      </div>
    </div>

    <canvas ref="canvas" class="maze-canvas"></canvas> <!-- холст для лабиринта -->

    <div class="button-container">
      <button @click="resetLevel" class="reset-btn">Сбросить уровень</button> <!-- кнопка сброса уровня -->
    </div>

    <p v-if="won" class="win-message">УРОВЕНЬ ПРОЙДЕН!</p> <!-- сообщение о победе -->
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'

const canvas = ref(null) // ссылка на canvas
const ctx = ref(null) // контекст рисования

const ROWS = 13 // количество рядов
const COLS = 13 // количество колонок
const CELL = 38 // размер клетки

const level = ref(0) // текущий уровень
const player = ref({ r: 1, c: 1 }) // позиция игрока
const maze = ref([]) // массив лабиринта
const painted = ref([]) // закрашенные клетки
const particles = ref([]) // частицы
const won = ref(false) // выигрыш
const time = ref(0) // таймер
let timerInterval

const generateMaze = () =>
{ // генерация лабиринта
  let newMaze = Array.from({ length: ROWS }, () => Array(COLS).fill(0)) // создаем пустую матрицу
  const visited = Array.from({ length: ROWS }, () => Array(COLS).fill(false)) // массив посещений

  const stack = [[1, 1]] // стек для dfs
  visited[1][1] = true
  newMaze[1][1] = 1

  const dirs = [[0, 2], [2, 0], [0, -2], [-2, 0]] // направления движения

  while (stack.length > 0) 
  { // основной цикл dfs
    const [r, c] = stack[stack.length - 1]
    let neighbors = []

    for (let [dr, dc] of dirs) 
    { // ищем непосещенных соседей
      const nr = r + dr
      const nc = c + dc
      if (nr > 0 && nr < ROWS - 1 && nc > 0 && nc < COLS - 1 && !visited[nr][nc]) 
        neighbors.push([nr, nc])
    }

    if (neighbors.length === 0) { stack.pop(); continue } // если соседей нет

    neighbors.sort(() => Math.random() - 0.5)
    const [nr, nc] = neighbors[0]

    newMaze[r + Math.floor((nr - r) / 2)][c + Math.floor((nc - c) / 2)] = 1 // прокладываем путь
    newMaze[nr][nc] = 1
    visited[nr][nc] = true
    stack.push([nr, nc])
  }

  for (let r = 1; r < ROWS - 1; r++) 
  { // дополнительная очистка
    for (let c = 1; c < COLS - 1; c++) 
    {
      if (newMaze[r][c] === 0) 
      {
        let adjacentPaths = 0
        const check = [[-1,0],[1,0],[0,-1],[0,1]]
        for (let [dr, dc] of check) 
          if (newMaze[r + dr] && newMaze[r + dr][c + dc] === 1) adjacentPaths++
        if (adjacentPaths >= 2 && Math.random() > 0.6) newMaze[r][c] = 1
      }
    }
  }

  newMaze[1][1] = 1 // стартовая клетка
  newMaze[ROWS - 2][COLS - 2] = 1 // выход

  return newMaze
}

class Particle 
{ // класс частицы
  constructor(x, y, opt = {}) 
  {
    this.x = x
    this.y = y
    this.vx = opt.vx ?? (Math.random() - 0.5) * 3
    this.vy = opt.vy ?? (Math.random() - 0.5) * 3
    this.life = opt.life ?? 32
    this.size = opt.size ?? 3.5
    this.color = opt.color ?? '#ff66ff'
  }

  update() { this.life--; this.x += this.vx; this.y += this.vy; this.vy += 0.08 
  } // обновление позиции

  draw(ctx) 
  { // рисуем частицу
    ctx.globalAlpha = this.life / 35
    ctx.fillStyle = this.color
    ctx.beginPath()
    ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2)
    ctx.fill()
    ctx.globalAlpha = 1
  }
}

const totalFloors = computed(() => maze.value.flat().filter(c => c === 1).length) // всего клеток
const paintedCount = computed(() => painted.value.flat().filter(Boolean).length) // закрашено

let t = 0
const draw = () => 
{ // отрисовка
  t += 0.055
  ctx.value.clearRect(0, 0, canvas.value.width, canvas.value.height)

  for (let r = 0; r < ROWS; r++) 
  {
    for (let c = 0; c < COLS; c++) 
    {
      const x = c * CELL
      const y = r * CELL

      if (maze.value[r][c] === 0) { ctx.value.fillStyle = '#1f1f1f'; ctx.value.fillRect(x, y, CELL, CELL) } // стена
      else 
      { // пол
        const isPainted = painted.value[r][c]
        const pulse = isPainted ? Math.sin(t * 5) * 6 : 0
        ctx.value.shadowBlur = isPainted ? 25 : 8
        ctx.value.shadowColor = isPainted ? '#ff66ff' : '#777'
        ctx.value.fillStyle = isPainted ? `rgb(185, 40, ${175 + pulse})` : '#bbbbbb'
        ctx.value.fillRect(x, y, CELL, CELL)
        ctx.value.shadowBlur = 0
      }
    }
  }

  drawBall(player.value, '#ff3366', t) // шарик игрока

  for (let i = particles.value.length - 1; i >= 0; i--) 
  { // частицы
    const p = particles.value[i]
    p.update()
    p.draw(ctx.value)
    if (p.life <= 0) particles.value.splice(i, 1)
  }
}

const drawBall = (obj, color, tick) =>
{ // рисуем шарик
  const x = obj.c * CELL + CELL / 2
  const y = obj.r * CELL + CELL / 2
  const bobY = Math.sin(tick * 3.5) * 2.8
  const bobX = Math.sin(tick * 2.2) * 1.6

  ctx.value.shadowBlur = 22; ctx.value.shadowColor = 'rgba(0,0,0,0.6)' // тень
  ctx.value.fillStyle = 'rgba(0,0,0,0.55)'
  ctx.value.beginPath()
  ctx.value.ellipse(x + 4, y + 11 + bobY, CELL/2 - 8, 7, 0, 0, Math.PI * 2)
  ctx.value.fill()

  ctx.value.shadowBlur = 35; ctx.value.shadowColor = color // шарик
  ctx.value.fillStyle = color
  ctx.value.beginPath()
  ctx.value.arc(x + bobX, y + bobY, CELL/2 - 7, 0, Math.PI * 2)
  ctx.value.fill()

  ctx.value.shadowBlur = 0; ctx.value.fillStyle = 'rgba(255,255,255,0.8)' // блик
  ctx.value.beginPath()
  ctx.value.arc(x - 8 + bobX, y - 9 + bobY, 7.5, 0, Math.PI * 2)
  ctx.value.fill()
}

const dirs = { ArrowUp: { dr: -1, dc: 0 }, ArrowDown: { dr: 1, dc: 0 }, ArrowLeft: { dr: 0, dc: -1 }, ArrowRight: { dr: 0, dc: 1 } } // направления движения

const move = async (key) => 
{ // движение игрока
  if (!dirs[key] || won.value) return
  let { dr, dc } = dirs[key]
  let { r, c } = player.value

  while (true) 
  {
    let nr = r + dr
    let nc = c + dc
    if (nr < 0 || nr >= ROWS || nc < 0 || nc >= COLS || maze.value[nr][nc] === 0) break

    r = nr; c = nc

    if (!painted.value[r][c]) 
    { // закрашиваем и создаем частицы
      painted.value[r][c] = true
      for (let i = 0; i < 14; i++) particles.value.push(new Particle(c * CELL + CELL / 2, r * CELL + CELL / 2, { color: '#ff77ff' }))
    }

    player.value = { r, c }
    await new Promise(res => setTimeout(res, 34))
  }

  if (paintedCount.value === totalFloors.value) { won.value = true; setTimeout(nextLevel, 850) } // проверка победы
}

const loadLevel = () => { maze.value = generateMaze(); painted.value = Array.from({ length: ROWS }, () => Array(COLS).fill(false)); player.value = { r: 1, c: 1 }; particles.value = []; won.value = false; startTimer() } // загрузка уровня
const nextLevel = () => { level.value++; loadLevel() } // следующий уровень
const resetLevel = () => { painted.value = Array.from({ length: ROWS }, () => Array(COLS).fill(false)); player.value = { r: 1, c: 1 }; particles.value = []; won.value = false; startTimer() } // сброс уровня

const startTimer = () => { clearInterval(timerInterval); time.value = 0; timerInterval = setInterval(() => time.value++, 1000) } // таймер
const loop = () => { draw(); requestAnimationFrame(loop) } // игровой цикл
const handleKey = e => move(e.key) // обработка клавиш

onMounted(() => { canvas.value.width = COLS * CELL; canvas.value.height = ROWS * CELL; ctx.value = canvas.value.getContext('2d'); loadLevel(); window.addEventListener('keydown', handleKey); loop() })
onUnmounted(() => { window.removeEventListener('keydown', handleKey); clearInterval(timerInterval) })
</script>

<style scoped>
.game-container 
{ text-align: center; background: #0f0f0f; color: #fff; padding: 30px 20px; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; min-height: 100vh } /* контейнер игры */

.header 
{ display: flex; align-items: center; justify-content: center; gap: 15px; margin-bottom: 10px } /* заголовок */

h1 
{ font-size: 2.25rem; font-weight: 700; color: #ff66ff; margin: 0; text-shadow: 0 0 18px #ff44ff } /* текст заголовка */

.stats 
{ display: flex; justify-content: center; gap: 50px; margin: 25px 0 35px 0; flex-wrap: wrap } /* статистика */

.stat-item 
{ display: flex; flex-direction: column; align-items: center; min-width: 130px } /* элемент статистики */

.label 
{ font-size: 1rem; color: #999; margin-bottom: 6px } /* подпись */

.value 
{ font-size: 1.65rem; font-weight: 700; color: #fff; text-shadow: 0 0 12px rgba(255, 102, 255, 0.5) } /* значение */

.maze-canvas 
{ border: 9px solid #333; border-radius: 18px; box-shadow: 0 15px 50px rgba(0, 0, 0, 0.9); background: #1a1a1a; margin: 15px auto 35px } /* холст лабиринта */

.reset-btn 
{ padding: 15px 36px; border: none; border-radius: 14px; background: linear-gradient(135deg, #ff66ff, #cc00cc); color: white; font-weight: 600; font-size: 1.15rem; cursor: pointer; box-shadow: 0 8px 25px rgba(255, 102, 255, 0.4); transition: all 0.25s } /* кнопка сброса */

.reset-btn:hover 
{ transform: translateY(-4px); box-shadow: 0 12px 30px rgba(255, 102, 255, 0.55) } /* эффект наведения */

.win-message 
{ font-size: 2.2rem; color: #00ffaa; margin-top: 30px; text-shadow: 0 0 25px #00ffaa; font-weight: 700; animation: winPulse 1.7s infinite } /* сообщение победы */

@keyframes winPulse 
{ 0%, 100% { transform: scale(1) } 50% { transform: scale(1.08) } }
</style>