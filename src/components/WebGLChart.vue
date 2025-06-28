<template>
  <div class="chart-container">
    <h3>WebGL Plot</h3>
    <canvas ref="webglCanvas" class="webgl-canvas"></canvas>
    <div class="legend">
      <div class="legend-item">
        <span class="legend-color" style="background-color: #e74c3c;"></span>
        <span>CPU Usage (%)</span>
      </div>
      <div class="legend-item">
        <span class="legend-color" style="background-color: #3498db;"></span>
        <span>Memory Usage (MB)</span>
      </div>
      <div class="legend-item">
        <span class="legend-color" style="background-color: #2ecc71;"></span>
        <span>Network I/O (KB/s)</span>
      </div>
      <div class="legend-item">
        <span class="legend-color" style="background-color: #f39c12;"></span>
        <span>Disk I/O (KB/s)</span>
      </div>
    </div>
  </div>
</template>

<script>
import { WebglPlot, WebglLine, ColorRGBA } from 'webgl-plot'

export default {
  name: 'WebGLChart',
  props: {
    data: {
      type: Object,
      required: true
    }
  },
  data() {
    return {
      webglPlot: null,
      lines: []
    }
  },
  mounted() {
    this.initChart()
  },
  watch: {
    data: {
      handler() {
        this.updateChart()
      },
      deep: true
    }
  },
  methods: {
    initChart() {
      const canvas = this.$refs.webglCanvas
      canvas.width = canvas.clientWidth
      canvas.height = 300
      
      this.webglPlot = new WebglPlot(canvas)
      
      const colors = [
        new ColorRGBA(0.9, 0.3, 0.23, 1), // #e74c3c
        new ColorRGBA(0.2, 0.6, 0.86, 1), // #3498db
        new ColorRGBA(0.18, 0.8, 0.44, 1), // #2ecc71
        new ColorRGBA(0.95, 0.61, 0.07, 1) // #f39c12
      ]
      
      for (let i = 0; i < 4; i++) {
        const line = new WebglLine(colors[i], 1000)
        this.lines.push(line)
        this.webglPlot.addLine(line)
      }
    },
    
    updateChart() {
      if (!this.webglPlot || !this.data.timestamps.length) return
      
      const startTime = performance.now()
      const memoryBefore = performance.memory ? performance.memory.usedJSHeapSize : 0
      
      const dataLength = this.data.timestamps.length
      const series = [this.data.series1, this.data.series2, this.data.series3, this.data.series4]
      
      // Normalize data for WebGL coordinates
      const maxValues = series.map(s => Math.max(...s))
      const minValues = series.map(s => Math.min(...s))
      
      for (let i = 0; i < 4; i++) {
        const line = this.lines[i]
        line.clear()
        
        const range = maxValues[i] - minValues[i] || 1
        
        for (let j = 0; j < dataLength; j++) {
          const x = (j / Math.max(dataLength - 1, 1)) * 2 - 1 // Normalize to [-1, 1]
          const y = ((series[i][j] - minValues[i]) / range) * 2 - 1 // Normalize to [-1, 1]
          line.addPoint(x, y)
        }
      }
      
      this.webglPlot.update()
      
      const endTime = performance.now()
      const memoryAfter = performance.memory ? performance.memory.usedJSHeapSize : 0
      
      this.$emit('performance', {
        renderTime: endTime - startTime,
        memoryUsage: (memoryAfter - memoryBefore) / 1024 / 1024,
        cpuUsage: Math.random() * 10 + 2 // Simulated CPU usage
      })
    }
  }
}
</script>

<style scoped>
.chart-container {
  padding: 20px;
  height: 400px;
}

.chart-container h3 {
  margin-bottom: 15px;
  color: #2c3e50;
  text-align: center;
}

.webgl-canvas {
  width: 100%;
  height: 300px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.legend {
  display: flex;
  justify-content: center;
  gap: 20px;
  margin-top: 10px;
  flex-wrap: wrap;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 12px;
}

.legend-color {
  width: 12px;
  height: 12px;
  border-radius: 2px;
}
</style>