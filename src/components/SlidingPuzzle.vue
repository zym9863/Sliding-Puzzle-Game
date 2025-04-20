<template>
  <div class="sliding-puzzle">
    <div class="controls">
      <div class="info">
    <div class="moves"><span class="icon-move">⬆️</span> 步数: {{ moves }}</div>
    <div class="timer"><span class="icon-timer">⏰</span> 时间: {{ formatTime(timer) }}</div>
      </div>
      <div class="buttons">
        <button @click="startNewGame" class="btn">新游戏</button>
        <button @click="togglePause" class="btn">{{ isPaused ? '继续' : '暂停' }}</button>
        <select v-model="gridSize" @change="startNewGame" class="select">
          <option value="3">3 x 3</option>
          <option value="4">4 x 4</option>
          <option value="5">5 x 5</option>
        </select>
      </div>
    </div>

    <div class="game-container" :class="{ 'paused': isPaused, 'completed': isCompleted }">
      <div v-if="isPaused" class="overlay">
        <div class="message">已暂停</div>
      </div>
      <div v-if="isCompleted" class="overlay">
        <div class="message">
          <h2>恭喜!</h2>
          <p>你完成了拼图!</p>
          <p>步数: {{ moves }}</p>
          <p>时间: {{ formatTime(timer) }}</p>
          <button @click="startNewGame" class="btn">再来一次</button>
        </div>
      </div>

      <div class="puzzle-grid" :style="gridStyle">
        <div 
          v-for="tile in tiles" 
          :key="tile.id"
          class="tile"
          :class="{ 'empty': tile.value === 0, 'correct': tile.isCorrect }"
          :style="getTileStyle(tile)"
          @click="moveTile(tile)"
        >
          <span v-if="tile.value !== 0">{{ tile.value }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'SlidingPuzzle',
  data() {
    return {
      gridSize: 4,
      tiles: [],
      emptyTileIndex: 0,
      moves: 0,
      timer: 0,
      timerInterval: null,
      isPaused: false,
      isCompleted: false,
      gameStarted: false
    }
  },
  computed: {
    gridStyle() {
      return {
        gridTemplateColumns: `repeat(${this.gridSize}, 1fr)`,
        gridTemplateRows: `repeat(${this.gridSize}, 1fr)`
      }
    }
  },
  mounted() {
    this.startNewGame()
  },
  beforeUnmount() {
    this.clearTimer()
  },
  methods: {
    startNewGame() {
      this.clearTimer()
      this.moves = 0
      this.timer = 0
      this.isPaused = false
      this.isCompleted = false
      this.gameStarted = false
      this.initializeTiles()
      this.shuffleTiles()
    },
    
    initializeTiles() {
      this.tiles = []
      const totalTiles = this.gridSize * this.gridSize
      
      for (let i = 0; i < totalTiles; i++) {
        this.tiles.push({
          id: i,
          value: i === totalTiles - 1 ? 0 : i + 1,
          row: Math.floor(i / this.gridSize),
          col: i % this.gridSize,
          isCorrect: false
        })
      }
      
      this.emptyTileIndex = totalTiles - 1
      this.updateCorrectPositions()
    },
    
    shuffleTiles() {
      // 执行足够多的随机移动来打乱拼图
      const moves = this.gridSize * this.gridSize * 20
      for (let i = 0; i < moves; i++) {
        const emptyTile = this.tiles[this.emptyTileIndex]
        const adjacentTiles = this.getAdjacentTiles(emptyTile)
        
        if (adjacentTiles.length > 0) {
          const randomIndex = Math.floor(Math.random() * adjacentTiles.length)
          const tileToMove = adjacentTiles[randomIndex]
          this.swapTiles(tileToMove, emptyTile, false)
        }
      }
      
      // 确保拼图是可解的
      if (!this.isSolvable()) {
        // 如果不可解，交换两个非空白的方块使其可解
        const firstTile = this.tiles.find(tile => tile.value !== 0)
        const secondTile = this.tiles.find(tile => tile.value !== 0 && tile.id !== firstTile.id)
        
        const temp = firstTile.value
        firstTile.value = secondTile.value
        secondTile.value = temp
      }
      
      this.updateCorrectPositions()
    },
    
    isSolvable() {
      // 检查拼图是否可解
      const values = this.tiles.map(tile => tile.value).filter(value => value !== 0)
      let inversions = 0
      
      for (let i = 0; i < values.length; i++) {
        for (let j = i + 1; j < values.length; j++) {
          if (values[i] > values[j]) {
            inversions++
          }
        }
      }
      
      // 对于奇数大小的网格，逆序数必须为偶数才可解
      if (this.gridSize % 2 === 1) {
        return inversions % 2 === 0
      } 
      // 对于偶数大小的网格，逆序数加上空白方块所在行数（从底部数）的奇偶性决定可解性
      else {
        const emptyTile = this.tiles[this.emptyTileIndex]
        const emptyRowFromBottom = this.gridSize - emptyTile.row
        return (inversions + emptyRowFromBottom) % 2 === 1
      }
    },
    
    getAdjacentTiles(tile) {
      return this.tiles.filter(t => {
        return (
          (Math.abs(t.row - tile.row) === 1 && t.col === tile.col) ||
          (Math.abs(t.col - tile.col) === 1 && t.row === tile.row)
        )
      })
    },
    
    moveTile(tile) {
      if (this.isPaused || this.isCompleted) return
      
      const emptyTile = this.tiles[this.emptyTileIndex]
      const isAdjacent = this.getAdjacentTiles(emptyTile).includes(tile)
      
      if (isAdjacent) {
        if (!this.gameStarted) {
          this.startTimer()
          this.gameStarted = true
        }
        
        this.swapTiles(tile, emptyTile, true)
        this.checkCompletion()
      }
    },
    
    swapTiles(tile1, tile2, countMove = true) {
      // 交换两个方块的值
      const tempValue = tile1.value
      tile1.value = tile2.value
      tile2.value = tempValue
      
      // 更新空白方块索引
      if (tile1.value === 0) {
        this.emptyTileIndex = this.tiles.findIndex(t => t.id === tile1.id)
      } else if (tile2.value === 0) {
        this.emptyTileIndex = this.tiles.findIndex(t => t.id === tile2.id)
      }
      
      if (countMove) {
        this.moves++
      }
      
      this.updateCorrectPositions()
    },
    
    updateCorrectPositions() {
      const totalTiles = this.gridSize * this.gridSize
      
      this.tiles.forEach(tile => {
        if (tile.value === 0) {
          tile.isCorrect = tile.id === totalTiles - 1
        } else {
          tile.isCorrect = tile.value === tile.id + 1
        }
      })
    },
    
    checkCompletion() {
      const isComplete = this.tiles.every(tile => tile.isCorrect)
      
      if (isComplete) {
        this.isCompleted = true
        this.clearTimer()
      }
    },
    
    startTimer() {
      this.clearTimer()
      this.timerInterval = setInterval(() => {
        if (!this.isPaused && !this.isCompleted) {
          this.timer++
        }
      }, 1000)
    },
    
    clearTimer() {
      if (this.timerInterval) {
        clearInterval(this.timerInterval)
        this.timerInterval = null
      }
    },
    
    togglePause() {
      if (this.isCompleted || !this.gameStarted) return
      
      this.isPaused = !this.isPaused
    },
    
    formatTime(seconds) {
      const mins = Math.floor(seconds / 60)
      const secs = seconds % 60
      return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`
    },
    
    getTileStyle(tile) {
      return {
        backgroundColor: tile.isCorrect ? '#4caf50' : '#3498db',
        opacity: tile.value === 0 ? 0 : 1
      }
    }
  }
}
</script>

<style scoped>
.sliding-puzzle {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: 0 auto;
  max-width: 500px;
}

.controls {
  width: 100%;
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.info {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  font-size: 18px;
}

.buttons {
  display: flex;
  gap: 10px;
}

.btn {
  padding: 10px 20px;
  background: linear-gradient(135deg, #6a11cb 0%, #2575fc 100%);
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  font-size: 16px;
  transition: all 0.3s;
  box-shadow: 0 4px 8px rgba(0,0,0,0.2);
}

.btn:hover {
  background: linear-gradient(135deg, #2575fc 0%, #6a11cb 100%);
  transform: translateY(-2px);
  box-shadow: 0 6px 12px rgba(0,0,0,0.3);
}

.select {
  padding: 8px;
  border-radius: 4px;
  border: 1px solid #ccc;
  font-size: 16px;
}

.game-container {
  position: relative;
  width: 100%;
  aspect-ratio: 1 / 1;
  margin: 0 auto;
  border: 2px solid #2c3e50;
  border-radius: 8px;
  overflow: hidden;
}

.puzzle-grid {
  display: grid;
  width: 100%;
  height: 100%;
  gap: 2px;
  background-color: #2c3e50;
}

.tile {
  display: flex;
  justify-content: center;
  align-items: center;
  background: linear-gradient(145deg, #00c6fb 0%, #005bea 100%);
  color: white;
  font-size: calc(16px + 1vw);
  font-weight: bold;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 4px 8px rgba(0,0,0,0.15);
}

.tile:hover:not(.empty) {
  transform: scale(0.95);
}

.tile.empty {
  background-color: transparent;
  cursor: default;
}

.tile.correct {
  background: linear-gradient(145deg, #38ef7d 0%, #11998e 100%);
}

.overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 10;
}

.message {
  background: linear-gradient(to bottom, #ffffff 0%, #f8f9fa 100%);
  padding: 30px;
  border-radius: 16px;
  text-align: center;
  color: #424242;
  box-shadow: 0 8px 24px rgba(0,0,0,0.1);
}

.message h2 {
  margin-bottom: 10px;
  color: #4caf50;
}

.message p {
  margin-bottom: 10px;
}

@media (max-width: 600px) {
  .controls {
    flex-direction: column;
    gap: 15px;
  }
  
  .info {
    width: 100%;
    flex-direction: row;
    justify-content: space-between;
  }
}
</style>
