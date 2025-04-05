<template>
  <div class="chess-container">
    <div class="board">
      <!-- Rank labels (1-8) on the left -->
      <div v-for="rank in ranks" :key="'rank-'+rank" class="rank-label left">
        {{ rank }}
      </div>
      
      <!-- File labels (a-h) on top -->
      <div v-for="file in files" :key="'file-top-'+file" class="file-label top">
        {{ file }}
      </div>

      <!-- Клетки доски -->
      <div v-for="(square, index) in board" :key="square.square" class="square"
        :class="[
          square.color, 
          selected === square.square ? 'selected' : '', 
          isValidMove(square.square) ? 'valid-move' : '',
          lastMove?.from === square.square || lastMove?.to === square.square ? 'last-move' : ''
        ]"
        @click="handleSquareClick(square.square)">
        <img v-if="square.piece" :src="getPieceImg(square.piece)" />
      </div>

      <!-- Метки рядов справа -->
      <div v-for="rank in ranks" :key="'rank-right-'+rank" class="rank-label right">
        {{ rank }}
      </div>
      
      <!-- Метки столбцов снизу -->
      <div v-for="file in files" :key="'file-bottom-'+file" class="file-label bottom">
        {{ file }}
      </div>
    </div>

    <div class="info">
      <p>Ход: {{ game.turn() === 'w' ? 'белые' : 'черные ' }}</p>
      <p v-if="game.isGameOver()" class="game-over">Игра окончена! {{ gameStatus }}</p>
      <p v-else-if="gameStatus" class="game-status">{{ gameStatus }}</p>
      
      <div class="controls">
        <button @click="undoMove" :disabled="moves.length === 0">Отменить ход</button>
        <button @click="resetGame">Новая игра</button>
      </div>
      
      <h3>История ходов</h3>
      <ul class="moves">
        <li v-for="(move, index) in moves" :key="index">
          {{ index % 2 === 0 ? `${Math.floor(index/2) + 1}. ${move}` : move }}
        </li>
      </ul>
    </div>

    <!-- Захваченные фигуры -->
    <div class="captured-pieces">
      <div class="captured white">
        <h4>Захваченные белые:</h4>
        <div class="pieces">
          <img
            v-for="(piece, index) in capturedPieces.white"
            :key="'white-captured-' + index"
            :src="getPieceImg('w' + piece.toLowerCase())"
            alt="Захваченная фигура"
          />
        </div>
      </div>

      <div class="captured black">
        <h4>Захваченные черные:</h4>
        <div class="pieces">
          <img
            v-for="(piece, index) in capturedPieces.black"
            :key="'black-captured-' + index"
            :src="getPieceImg('b' + piece.toLowerCase())"
            alt="Захваченная фигура"
          />
        </div>
      </div>
    </div>

    <!-- Promotion modal -->
    <div v-if="showPromotionDialog" class="promotion-dialog">
      <div class="promotion-options">
        <h3>Выберите фигуру:</h3>
        <div v-for="piece in promotionPieces" :key="piece" @click="completePromotion(piece)">
          <img :src="getPieceImg(pendingPromotion.color + piece)" />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed } from 'vue'
import { Chess } from 'chess.js'

export default {
  setup() {
    const game = ref(new Chess())
    const selected = ref(null)
    const moves = ref([])
    const validMoves = ref([])
    const lastMove = ref(null)

    const files = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h']
    const ranks = [8, 7, 6, 5, 4, 3, 2, 1] // Изменён порядок для правильного отображения

    const currentPlayer = computed(() => {
      return game.value.turn() === 'w' ? 'белые' : 'черные'
    })

    const capturedPieces = ref({
      white: [],
      black: []
    })

    const board = computed(() => {
      const squares = []
      const positions = game.value.board()
      
      // Правильный порядок обхода доски (a8 -> h1)
      for (let row = 0; row < 8; row++) {
        for (let col = 0; col < 8; col++) {
          const square = files[col] + ranks[row]
          const piece = positions[row]?.[col]
          squares.push({
            square,
            color: (row + col) % 2 === 0 ? 'black' : 'white', // Инвертированы цвета
            piece: piece ? piece.color + piece.type : null
          })
        }
      }
      return squares
    })

    const getPieceImg = (piece) => {
      return `/pieces/${piece}.svg`
    }

    const getValidMoves = (square) => {
      return game.value.moves({
        square,
        verbose: true
      }).map(move => move.to)
    }

    const isValidMove = (square) => {
      return validMoves.value.includes(square)
    }

    const handleSquareClick = (square) => {
      if (selected.value) {
        try {
          const move = game.value.move({
            from: selected.value,
            to: square,
            promotion: 'q' // Всегда превращаем в ферзя для простоты
          });
          
          if (move) {
            moves.value.push(move.san)
            lastMove.value = { from: move.from, to: move.to }

            // Проверка на захват фигуры
            if (move.captured) {
              const capturedPiece = move.captured.toUpperCase();
              const color = move.color === 'w' ? 'black' : 'white';
              capturedPieces.value[color].push(capturedPiece);
            }
          }
        } catch (e) {
          console.log("Недопустимый ход")
        }
        
        selected.value = null
        validMoves.value = []
      } 
      else {
        const piece = game.value.get(square)
        if (piece && piece.color === game.value.turn()) {
          selected.value = square
          validMoves.value = getValidMoves(square)
        }
      }
    }

    const resetGame = () => {
      game.value = new Chess()
      selected.value = null
      moves.value = []
      validMoves.value = []
      lastMove.value = null
      capturedPieces.value = { white: [], black: [] }
    }

    return {
      game,
      board,
      selected,
      currentPlayer,
      moves,
      lastMove,
      files,
      ranks,
      handleSquareClick,
      getPieceImg,
      isValidMove,
      resetGame,
      capturedPieces
    }
  }
}
</script>

<style scoped>
.chess-container {
  display: flex;
  gap: 30px;
  justify-content: center;
  padding: 30px;
  flex-wrap: wrap;
  font-family: Arial, sans-serif;
}

.board {
  display: grid;
  grid-template-columns: repeat(8, 1fr);
  grid-template-rows: repeat(8, 1fr);
  border: 2px solid #222;
  width: 600px;
  height: 600px;
  position: relative;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.5);
}

.square {
  width: 100%;
  height: 100%;
  border: 1px solid #333;
  position: relative;
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
}

.white {
  background-color: #f0d9b5;
}

.black {
  background-color: #b58863;
}

.selected {
  background-color: rgba(255, 255, 0, 0.4) !important;
}

.valid-move {
  background-color: rgba(0, 255, 0, 0.719);
  opacity: 0.3;
}

.last-move {
}

.square img {
  width: 80%;
  height: 80%;
  object-fit: contain;
  pointer-events: none;
}

.info {
  color: #333;
  max-width: 300px;
  background: #f8f8f8;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.game-over {
  color: #d32f2f;
  font-weight: bold;
}

.game-status {
  color: #1976d2;
  font-weight: bold;
}

.controls {
  display: flex;
  gap: 10px;
  margin: 15px 0;
}

button {
  padding: 8px 15px;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background 0.3s;
}

button:hover {
  background: #388E3C;
}

button:disabled {
  background: #cccccc;
  cursor: not-allowed;
}

.moves {
  list-style: none;
  padding: 0;
  font-family: monospace;
  font-size: 14px;
  max-height: 500px;
  overflow-y: auto;
  background: white;
  padding: 10px;
  border-radius: 4px;
  border: 1px solid #ddd;
}

.moves li {
  padding: 3px 0;
  border-bottom: 1px solid #eee;
}

.moves li:last-child {
  border-bottom: none;
}

/* Label positioning */
.file-label, .rank-label {
  position: absolute;
  font-weight: bold;
  font-size: 16px;
  color: #333;
  user-select: none;
}

/* File labels (a-h) */
.file-label.top {
  top: -25px;
  left: 50%;
  transform: translateX(-50%);
}

.file-label.bottom {
  bottom: -25px;
  left: 50%;
  transform: translateX(-50%);
}

/* Rank labels (1-8) */
.rank-label.left {
  left: -25px;
  top: 50%;
  transform: translateY(-50%);
}

.rank-label.right {
  right: -25px;
  top: 50%;
  transform: translateY(-50%);
}

/* Promotion dialog */
.promotion-dialog {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 100;
}

.promotion-options {
  background: white;
  padding: 20px;
  border-radius: 8px;
  text-align: center;
}

.promotion-options h3 {
  margin-top: 0;
}

.promotion-options div {
  display: inline-block;
  margin: 10px;
  cursor: pointer;
  padding: 10px;
  border-radius: 4px;
  transition: background 0.3s;
}

.promotion-options div:hover {
  background: #f0f0f0;
}

.promotion-options img {
  width: 60px;
  height: 60px;
}

/* Captured pieces */
.captured-pieces {
  display: flex;
  flex-direction: column;
  gap: 20px;
  margin-top: 20px;
}

.captured {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.captured h4 {
  margin-bottom: 10px;
  font-size: 16px;
  font-weight: bold;
}

.captured .pieces {
  display: flex;
  gap: 5px;
}

.captured img {
  width: 30px;
  height: 30px;
  object-fit: contain;
}

@media (max-width: 768px) {
  .board {
    width: 90vw;
    height: 90vw;
  }
  
  .chess-container {
    flex-direction: column;
    align-items: center;
  }
  
  .info {
    max-width: 90%;
  }
}
</style>