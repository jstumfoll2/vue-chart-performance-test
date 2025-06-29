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
      lines: [],
      maxPoints: 1000
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
        const line = new WebglLine(colors[i], this.maxPoints)
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
      
      // Calculate global min/max for consistent scaling across all series
      const allValues = series.flat()
      const globalMax = Math.max(...allValues)
      const globalMin = Math.min(...allValues)
      const range = globalMax - globalMin || 1
      
      for (let i = 0; i < 4; i++) {
        const line = this.lines[i]
        
        // Limit data points to prevent buffer overflow
        const pointsToShow = Math.min(dataLength, this.maxPoints)
        const startIndex = Math.max(0, dataLength - pointsToShow)
        
        // Clear the entire xy array to prevent ghost points
        for (let k = 0; k < line.xy.length; k++) {
          line.xy[k] = 0
        }
        
        // Set the number of points for this line
        line.numPoints = pointsToShow
        
        for (let j = 0; j < pointsToShow; j++) {
          const dataIndex = startIndex + j
          
          // Normalize X coordinate to [-1, 1] based on time progression
          const x = (j / Math.max(pointsToShow - 1, 1)) * 2 - 1
          
          // Normalize Y coordinate to [-1, 1] based on value range
          const y = ((series[i][dataIndex] - globalMin) / range) * 2 - 1
          
          // Directly set coordinates in the line's data array
          line.xy[2 * j] = x      // X coordinate
          line.xy[2 * j + 1] = y  // Y coordinate
        }
        
        // Force buffer update
        line.webglNumPoints = pointsToShow
      }
      
      // Render the plot
      this.webglPlot.update()
      
      const endTime = performance.now()
      const memoryAfter = performance.memory ? performance.memory.usedJSHeapSize : 0
      
      this.$emit('performance', {
        renderTime: endTime - startTime,
        memoryUsage: (memoryAfter) / 1024 / 1024,
        cpuUsage: Math.random() * 10 + 2 // Simulated CPU usage
      })
    }
  },
  
  beforeUnmount() {
    // Clean up WebGL resources
    if (this.webglPlot) {
      this.webglPlot = null
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