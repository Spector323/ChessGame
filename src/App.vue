<template>
  <div class="app">
    <h1>♟ Онлайн шахматы</h1>
    <ChessGame />
  </div>
</template>

<script>
import ChessGame from './components/ChessGame.vue'

export default {
  components: {
    ChessGame
  }
}
</script>

<style scoped>
.app {
  text-align: center;
  padding: 20px;
  background: linear-gradient(to right, #1e1e1e, #2a2a2a);
  color: #fff;
  min-height: 100vh;
  font-family: 'Segoe UI', sans-serif;
}
h1 {
  margin-bottom: 30px;
}
</style>
