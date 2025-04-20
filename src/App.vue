<template>
  <div class="app-container">
    <h1>华容道滑块拼图</h1>
    <SlidingPuzzle />
  </div>
</template>

<script>
import SlidingPuzzle from './components/SlidingPuzzle.vue'

export default {
  name: 'App',
  components: {
    SlidingPuzzle
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Microsoft YaHei', Arial, sans-serif;
  background-color: #f5f5f5;
  color: #333;
}

.app-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  text-align: center;
}

h1 {
  margin-bottom: 20px;
  color: #2c3e50;
}
</style>
