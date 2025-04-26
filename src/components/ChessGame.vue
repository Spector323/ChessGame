<template>
  <div class="chess-container">
    <!-- Шахматная доска -->
    <div class="board-wrapper">
      <!-- Метки рядов слева -->
      <div v-for="rank in ranks" :key="'rank-left-' + rank" class="rank-label left">{{ rank }}</div>
      <!-- Метки столбцов сверху -->
      <div v-for="file in files" :key="'file-top-' + file" class="file-label top">{{ file }}</div>

      <!-- Клетки доски -->
      <div v-for="square in board" :key="square.square" class="square" :class="[
        square.color,
        selectedSquare === square.square ? 'selected' : '',
        isValidMove(square.square) ? 'valid-move' : '',
        isLastMove(square.square) ? 'last-move' : ''
      ]" @click="handleSquareClick(square.square)">
        <img v-if="square.piece" :src="getPieceImage(square.piece)" alt="Chess piece" />
      </div>

      <!-- Метки рядов справа -->
      <div v-for="rank in ranks" :key="'rank-right-' + rank" class="rank-label right">{{ rank }}</div>
      <!-- Метки столбцов снизу -->
      <div v-for="file in files" :key="'file-bottom-' + file" class="file-label bottom">{{ file }}</div>
    </div>

    <!-- Информация об игре -->
    <div class="game-info">
      <p>Ход: {{ currentPlayer }}</p>
      <p v-if="isGameOver" class="game-over">Игра окончена! {{ gameStatus }}</p>
      <p v-else-if="gameStatus" class="game-status">{{ gameStatus }}</p>

      <!-- Управление игрой -->
      <div class="controls">
        <button @click="undoMove" :disabled="!moveHistory.length">Отменить ход</button>
        <button @click="startNewGame">Новая игра</button>
      </div>

      <!-- История ходов -->
      <h3>История ходов</h3>
      <ul class="move-history">
        <li v-for="(move, index) in formattedMoveHistory" :key="index">
          {{ move }}
        </li>
      </ul>
    </div>

    <!-- Захваченные фигуры -->
    <div class="captured-pieces">
      <div class="captured white">
        <h4>Захваченные белые:</h4>
        <div class="pieces">
          <img v-for="(piece, index) in capturedPieces.white" :key="'white-captured-' + index"
            :src="getPieceImage('w' + piece.toLowerCase())" alt="Captured piece" />
        </div>
      </div>
      <div class="captured black">
        <h4>Захваченные черные:</h4>
        <div class="pieces">
          <img v-for="(piece, index) in capturedPieces.black" :key="'black-captured-' + index"
            :src="getPieceImage('b' + piece.toLowerCase())" alt="Captured piece" />
        </div>
      </div>
    </div>

    <!-- Диалог превращения пешки -->
    <div v-if="showPromotionDialog" class="promotion-dialog">
      <div class="promotion-options">
        <h3>Выберите фигуру:</h3>
        <div v-for="piece in promotionPieces" :key="piece" @click="completePromotion(piece)" class="promotion-piece">
          <img :src="getPieceImage(pendingPromotion.color + piece)" alt="Promotion piece" />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed } from 'vue';
import { Chess } from 'chess.js';

export default {
  name: 'ChessGame',
  setup() {
    const game = ref(new Chess());
    const selectedSquare = ref(null);
    const validMoves = ref([]);
    const moveHistory = ref([]);
    const lastMove = ref(null);
    const capturedPieces = ref({ white: [], black: [] });
    const showPromotionDialog = ref(false);
    const pendingPromotion = ref(null);
    const gameStatus = ref('');

    const files = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h'];
    const ranks = [8, 7, 6, 5, 4, 3, 2, 1];
    const promotionPieces = ['q', 'r', 'b', 'n'];

    const pieceValues = { p: 1, n: 3, b: 3, r: 5, q: 9, k: 0 };

    const board = computed(() => {
      const positions = game.value.board();
      const squares = [];

      for (let row = 0; row < 8; row++) {
        for (let col = 0; col < 8; col++) {
          const piece = positions[row]?.[col];
          squares.push({
            square: files[col] + ranks[row],
            piece: piece ? piece.color + piece.type : null,
            color: (row + col) % 2 === 0 ? 'black' : 'white',
          });
        }
      }
      return squares;
    });

    const currentPlayer = computed(() => game.value.turn() === 'w' ? 'белые' : 'черные');

    const formattedMoveHistory = computed(() => {
      return moveHistory.value.reduce((acc, move, idx) => {
        if (idx % 2 === 0) {
          acc.push(`${Math.floor(idx / 2) + 1}. ${move}`);
        } else {
          acc[acc.length - 1] += ` ${move}`;
        }
        return acc;
      }, []);
    });

    const getPieceImage = (piece) => `/pieces/${piece}.svg`;
    const isValidMove = (square) => validMoves.value.includes(square);
    const isLastMove = (square) => lastMove.value?.from === square || lastMove.value?.to === square;


    const handleSquareClick = (square) => {
      if (showPromotionDialog.value) return;

      if (selectedSquare.value) {
        const move = { from: selectedSquare.value, to: square };
        const piece = game.value.get(selectedSquare.value);

        // Проверка превращения пешки
        if (piece?.type === 'p' && ((piece.color === 'w' && square[1] === '8') || (piece.color === 'b' && square[1] === '1'))) {
          pendingPromotion.value = { ...move, color: piece.color };
          showPromotionDialog.value = true;
          return;
        }

        makeMove(move);
      } else {
        const piece = game.value.get(square);
        if (piece && piece.color === game.value.turn()) {
          selectedSquare.value = square;
          validMoves.value = game.value.moves({ square, verbose: true }).map(m => m.to);
        }
      }
    };

    const makeMove = (move, promotion = 'q') => {
      try {
        const executedMove = game.value.move({ ...move, promotion });
        if (executedMove) {
          updateState(executedMove);

          // Ход бота после игрока
          if (game.value.turn() === 'b' && !game.value.isGameOver()) {
            setTimeout(() => {
              makeBotMove();
            }, 700);
          }
        }
      } catch (error) {
        console.log('Ошибка хода:', error);
      }
      selectedSquare.value = null;
      validMoves.value = [];
    };

    const completePromotion = (piece) => {
      if (pendingPromotion.value) {
        makeMove({ ...pendingPromotion.value }, piece);
        pendingPromotion.value = null;
        showPromotionDialog.value = false;
      }
    };

    const updateState = (move) => {
      moveHistory.value.push(move.san);
      lastMove.value = { from: move.from, to: move.to };

      if (move.captured) {
        const captured = move.captured.toUpperCase();
        const color = move.color === 'w' ? 'black' : 'white';
        capturedPieces.value[color].push(captured);
      }

      updateGameStatus();
    };

    const updateGameStatus = () => {
      if (game.value.isCheckmate()) {
        gameStatus.value = `Шах и мат! Победили ${game.value.turn() === 'w' ? 'черные' : 'белые'}`;
      } else if (game.value.isStalemate()) {
        gameStatus.value = 'Пат! Ничья';
      } else if (game.value.isThreefoldRepetition()) {
        gameStatus.value = 'Трехкратное повторение — ничья';
      } else if (game.value.isInsufficientMaterial()) {
        gameStatus.value = 'Недостаточно материала — ничья';
      } else if (game.value.isDraw()) {
        gameStatus.value = 'Ничья по 50 ходам';
      } else if (game.value.inCheck()) {
        gameStatus.value = 'Шах!';
      } else {
        gameStatus.value = '';
      }
    };

    const startNewGame = () => {
      game.value.reset();
      moveHistory.value = [];
      selectedSquare.value = null;
      validMoves.value = [];
      lastMove.value = null;
      capturedPieces.value = { white: [], black: [] };
      pendingPromotion.value = null;
      showPromotionDialog.value = false;
      gameStatus.value = '';
    };

    const undoMove = () => {
      game.value.undo();
      moveHistory.value.pop();

      // Если отменяем ход игрока и бота сразу
      if (game.value.turn() === 'b') {
        game.value.undo();
        moveHistory.value.pop();
      }

      updateGameStatus();
    };

    const makeBotMove = () => {
      const moves = game.value.moves({ verbose: true });
      if (!moves.length || game.value.isGameOver()) return;

      const captures = moves.filter(m => m.captured);
      const bestMove = captures.length
        ? captures.reduce((best, move) => (pieceValues[move.captured] > pieceValues[best.captured] ? move : best), captures[0])
        : moves[Math.floor(Math.random() * moves.length)];

      makeMove({ from: bestMove.from, to: bestMove.to }, bestMove.promotion || 'q');
    };

    return {
      board,
      currentPlayer,
      selectedSquare,
      validMoves,
      lastMove,
      moveHistory,
      formattedMoveHistory,
      capturedPieces,
      showPromotionDialog,
      pendingPromotion,
      gameStatus,
      files,
      ranks,
      promotionPieces,
      getPieceImage,
      isValidMove,
      isLastMove,
      handleSquareClick,
      completePromotion,
      startNewGame,
      undoMove,
    };
  }
};
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

.board-wrapper {
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
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
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
  background-color: rgba(0, 255, 0, 0.3) !important;
}

.last-move {
  background-color: rgba(255, 165, 0, 0.3) !important;
}

.square img {
  width: 80%;
  height: 80%;
  object-fit: contain;
  pointer-events: none;
  transition: transform 0.3s ease;
  /* Добавляем плавное движение */
}


.game-info {
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

.move-history {
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

.move-history li {
  padding: 3px 0;
  border-bottom: 1px solid #eee;
}

.move-history li:last-child {
  border-bottom: none;
}

.file-label,
.rank-label {
  position: absolute;
  font-weight: bold;
  font-size: 16px;
  color: #333;
  user-select: none;
}

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

.promotion-piece {
  display: inline-block;
  margin: 10px;
  cursor: pointer;
  padding: 10px;
  border-radius: 4px;
  transition: background 0.3s;
}

.promotion-piece:hover {
  background: #f0f0f0;
}

.promotion-piece img {
  width: 60px;
  height: 60px;
}

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
  .board-wrapper {
    width: 90vw;
    height: 90vw;
  }

  .chess-container {
    flex-direction: column;
    align-items: center;
  }

  .game-info {
    max-width: 90%;
  }
}
</style>