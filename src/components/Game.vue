<template>
  <div>
    <div class="text-center mb-4">
      <h1 class="text-3xl font-bold text-yellow-400">CAT-PAC</h1>
      <p class="text-xl">Score: {{ score }}</p>
    </div>
    <div v-if="!gameWon && !gameOver" class="border-4 border-blue-500 p-2">
      <div 
        v-for="(row, rowIndex) in board" 
        :key="rowIndex" 
        class="flex"
      >
        <div 
          v-for="(cell, colIndex) in row" 
          :key="colIndex" 
          class="w-8 h-8 flex items-center justify-center"
        >
          <!-- Стены -->
          <div v-if="cell === 1" class="w-full h-full bg-blue-700 rounded-sm"></div>
          
          <!-- Еда -->
          <div v-if="cell === 2" class="w-2 h-2 bg-yellow-400 rounded-full"></div>

          <!-- Котопакмен -->
          <div v-if="cell === 3">
            <i class="fas fa-cat text-yellow-400 text-2xl"></i>
          </div>

          <!-- Призраки -->
          <div v-if="cell === 4">
            <i class="fas fa-ghost text-red-500 text-2xl animate__animated animate__pulse animate__infinite"></i>
          </div>
          <div v-if="cell === 5">
            <i class="fas fa-ghost text-pink-500 text-2xl animate__animated animate__pulse animate__infinite"></i>
          </div>
          <div v-if="cell === 6">
            <i class="fas fa-ghost text-cyan-500 text-2xl animate__animated animate__pulse animate__infinite"></i>
          </div>

        </div>
      </div>
    </div>
    <div v-else-if="gameWon" class="text-center">
      <h2 class="text-4xl font-bold text-green-500 animate__animated animate__tada">You Won!</h2>
      <p class="mt-2 text-lg">You ate all the pellets!</p>
      <button @click="restartGame" class="mt-4 px-4 py-2 bg-blue-500 hover:bg-blue-700 text-white font-bold rounded">
        Play Again
      </button>
    </div>
    <div v-else-if="gameOver" class="text-center">
      <h2 class="text-4xl font-bold text-red-500 animate__animated animate__shakeX">Game Over!</h2>
      <p class="mt-2 text-lg">A ghost caught you!</p>
      <button @click="restartGame" class="mt-4 px-4 py-2 bg-blue-500 hover:bg-blue-700 text-white font-bold rounded">
        Restart
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed } from 'vue'

// 0 - пустое место
// 1 - стена
// 2 - еда
// 3 - игрок
// 4 - призрак (красный)
// 5 - призрак (розовый)
// 6 - призрак (голубой)
const board = ref([
  [1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
  [1, 3, 2, 2, 2, 2, 2, 2, 2, 1],
  [1, 2, 1, 1, 2, 1, 1, 2, 5, 1],
  [1, 2, 1, 2, 2, 2, 1, 2, 2, 1],
  [1, 2, 2, 2, 1, 4, 2, 2, 2, 1],
  [1, 2, 1, 2, 2, 2, 1, 2, 2, 1],
  [1, 2, 1, 1, 2, 1, 1, 2, 2, 1],
  [1, 2, 2, 2, 6, 2, 2, 2, 2, 1],
  [1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
])
const initialBoard = JSON.parse(JSON.stringify(board.value))

// Позиция игрока
const player = ref({ x: 1, y: 1 })
const score = ref(0)
const gameWon = ref(false)
const gameOver = ref(false)
let gameInterval: number | undefined = undefined

interface Ghost {
  x: number;
  y: number;
  type: number; // 4, 5, or 6
  // Что было на клетке под призраком (0 - пусто, 2 - еда)
  underlyingCell: number; 
}

const ghosts = ref<Ghost[]>([])

// Считаем общее количество еды на поле
const totalPellets = computed(() => {
  return board.value.flat().filter(cell => cell === 2).length
})

const initializePositions = () => {
  ghosts.value = [] // Очищаем старых призраков
  board.value.forEach((row, y) => {
    row.forEach((cell, x) => {
      if (cell === 3) {
        player.value = { x, y }
      }
      if (cell >= 4) {
        ghosts.value.push({ x, y, type: cell, underlyingCell: 0 })
      }
    })
  })
}

const restartGame = () => {
  score.value = 0
  gameWon.value = false
  gameOver.value = false
  board.value = JSON.parse(JSON.stringify(initialBoard))
  initializePositions()

  clearInterval(gameInterval)
  gameInterval = window.setInterval(moveGhosts, 800)
}

// Находим начальные позиции игрока и призраков
// board.value.forEach((row, y) => {
//   row.forEach((cell, x) => {
//     if (cell === 3) {
//       player.value = { x, y }
//     }
//     if (cell >= 4) { // Призраки это 4, 5, 6
//       ghosts.value.push({ x, y, type: cell, underlyingCell: 0 })
//     }
//   })
// })

const moveGhosts = () => {
  if (gameWon.value || gameOver.value) return

  ghosts.value.forEach(ghost => {
    const { x, y } = ghost
    const possibleMoves: { x: number; y: number }[] = []

    const directions = [
      { x: x, y: y - 1 }, // Up
      { x: x, y: y + 1 }, // Down
      { x: x - 1, y: y }, // Left
      { x: x + 1, y: y }  // Right
    ]

    directions.forEach(dir => {
      const cellValue = board.value[dir.y]?.[dir.x]
      // Ходить можно если не стена (1) и не другой призрак (>= 4)
      if (cellValue !== undefined && cellValue !== 1 && cellValue < 4) {
        possibleMoves.push(dir)
      }
    })

    if (possibleMoves.length > 0) {
      const move = possibleMoves[Math.floor(Math.random() * possibleMoves.length)]!

      // Восстанавливаем клетку, где был призрак
      board.value[y]![x] = ghost.underlyingCell
      
      const destinationCell = board.value[move.y]![move.x]

      // Проверка на столкновение с игроком
      if (destinationCell === 3) {
        gameOver.value = true
        // Если игрок пойман, под призраком будет пустое место
        ghost.underlyingCell = 0
      } else {
        // Сохраняем, что находится на новой клетке
        if (destinationCell !== undefined) {
          ghost.underlyingCell = destinationCell
        }
      }
      
      // Перемещаем призрака
      board.value[move.y]![move.x] = ghost.type
      ghost.x = move.x
      ghost.y = move.y
    }
  })
}

const handleKeyDown = (e: KeyboardEvent) => {
  if (gameWon.value || gameOver.value) return // Если игра окончена, ничего не делаем

  const { x, y } = player.value
  let nextX = x
  let nextY = y

  switch (e.key) {
    case 'ArrowUp':
      nextY--
      break
    case 'ArrowDown':
      nextY++
      break
    case 'ArrowLeft':
      nextX--
      break
    case 'ArrowRight':
      nextX++
      break
    default:
      return // Ничего не делаем для других клавиш
  }

  // Проверка на столкновение со стеной
  const destinationCell = board.value[nextY]?.[nextX]
  if (destinationCell !== undefined && destinationCell !== 1) {
    // Проверяем, съели ли мы еду
    if (destinationCell === 2) {
      score.value += 10
    }
    // Проверка на столкновение с призраком после хода
    if (destinationCell >= 4) { // Призраки 4, 5, 6
      gameOver.value = true
      return // Завершаем ход
    }

    // Убираем игрока со старого места
    board.value[y]![x] = 0 // Становится пустым местом
    
    // Обновляем координаты
    player.value = { x: nextX, y: nextY }
    
    // Ставим игрока на новое место
    board.value[nextY]![nextX] = 3

    // Проверяем условие победы
    if (totalPellets.value === 0) {
      gameWon.value = true
    }
  }
}

onMounted(() => {
  initializePositions()
  window.addEventListener('keydown', handleKeyDown)
  gameInterval = window.setInterval(moveGhosts, 800) // Призраки двигаются каждые 800мс
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown)
  clearInterval(gameInterval)
})
</script>
