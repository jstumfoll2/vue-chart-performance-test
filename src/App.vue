<template>
  <div id="app">
    <header class="header">
      <h1>Vue Chart Libraries Performance Test</h1>
      <div class="controls">
        <button @click="toggleStreaming" :class="{ active: isStreaming }">
          {{ isStreaming ? 'Stop' : 'Start' }} Streaming
        </button>
        <button @click="clearData">Clear Data</button>
        <div class="speed-control">
          <label>Speed: </label>
          <select v-model="updateInterval">
            <option value="4">240 FPS (4ms)</option>
            <option value="8">120 FPS (8ms)</option>
            <option value="16">60 FPS (16ms)</option>
            <option value="33">30 FPS (33ms)</option>
            <option value="100">10 FPS (100ms)</option>
            <option value="500">2 FPS (500ms)</option>
          </select>
        </div>
      </div>
    </header>

    <div class="stats">
      <div class="stat">
        <span>Data Points per Series: {{ dataPoints }}</span>
      </div>
      <div class="stat">
        <span>Total Points: {{ dataPoints * 4 }}</span>
      </div>
      <div class="stat">
        <span>FPS: {{ currentFPS }}</span>
      </div>
    </div>


    <div class="performance-table">
      <h3>Performance Metrics</h3>
      <table>
        <thead>
          <tr>
            <th>Library</th>
            <th>Render Time (ms)</th>
            <th>Memory Usage (MB)</th>
            <th>CPU Usage (%)</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(perf, library) in performance" :key="library">
            <td>{{ library }}</td>
            <td>{{ perf.renderTime.toFixed(2) }}</td>
            <td>{{ perf.memoryUsage.toFixed(2) }}</td>
            <td>{{ perf.cpuUsage.toFixed(2) }}</td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
  
    <div class="charts-container">
      <div class="chart-wrapper">
        <ChartJsChart 
          :data="chartData" 
          @performance="updatePerformance('chartjs', $event)"
        />
      </div>
      
      <div class="chart-wrapper">
        <PlotlyChart 
          :data="chartData" 
          @performance="updatePerformance('plotly', $event)"
        />
      </div>
      
      <div class="chart-wrapper">
        <UplotChart 
          :data="chartData" 
          @performance="updatePerformance('uplot', $event)"
        />
      </div>
      
      <div class="chart-wrapper">
        <WebGLChart 
          :data="chartData" 
          @performance="updatePerformance('webgl', $event)"
        />
      </div>
    </div>
</template>

<script>
import ChartJsChart from './components/ChartJsChart.vue'
import PlotlyChart from './components/PlotlyChart.vue'
import UplotChart from './components/UplotChart.vue'
import WebGLChart from './components/WebGLChart.vue'

export default {
  name: 'App',
  components: {
    ChartJsChart,
    PlotlyChart,
    UplotChart,
    WebGLChart
  },
  data() {
    return {
      isStreaming: false,
      updateInterval: 100,
      intervalId: null,
      dataPoints: 0,
      currentFPS: 0,
      lastFrameTime: 0,
      chartData: {
        timestamps: [],
        series1: [], // CPU Usage
        series2: [], // Memory Usage
        series3: [], // Network I/O
        series4: []  // Disk I/O
      },
      performance: {
        chartjs: { renderTime: 0, memoryUsage: 0, cpuUsage: 0 },
        plotly: { renderTime: 0, memoryUsage: 0, cpuUsage: 0 },
        uplot: { renderTime: 0, memoryUsage: 0, cpuUsage: 0 },
        webgl: { renderTime: 0, memoryUsage: 0, cpuUsage: 0 }
      }
    }
  },
  methods: {
    toggleStreaming() {
      if (this.isStreaming) {
        this.stopStreaming()
      } else {
        this.startStreaming()
      }
    },
    
    startStreaming() {
      this.isStreaming = true
      this.lastFrameTime = performance.now()
      
      this.intervalId = setInterval(() => {
        this.addDataPoint()
        this.calculateFPS()
      }, parseInt(this.updateInterval))
    },
    
    stopStreaming() {
      this.isStreaming = false
      if (this.intervalId) {
        clearInterval(this.intervalId)
        this.intervalId = null
      }
    },
    
    addDataPoint() {
      const now = Date.now()
      const timestamp = new Date(now)
      
      // Compare render times
      this.chartData.timestamps.push(timestamp)
      this.chartData.series1.push(this.performance.chartjs.renderTime)
      this.chartData.series2.push(this.performance.plotly.renderTime)
      this.chartData.series3.push(this.performance.uplot.renderTime)
      this.chartData.series4.push(this.performance.webgl.renderTime)
      
      // Keep only last 1000 points to prevent memory issues
      const maxPoints = 10000
      if (this.chartData.timestamps.length > maxPoints) {
        this.chartData.timestamps.shift()
        this.chartData.series1.shift()
        this.chartData.series2.shift()
        this.chartData.series3.shift()
        this.chartData.series4.shift()
      }
      
      this.dataPoints = this.chartData.timestamps.length
    },
    
    clearData() {
      this.chartData = {
        timestamps: [],
        series1: [],
        series2: [],
        series3: [],
        series4: []
      }
      this.dataPoints = 0
    },
    
    calculateFPS() {
      const currentTime = performance.now()
      const deltaTime = currentTime - this.lastFrameTime
      this.currentFPS = Math.round(1000 / deltaTime)
      this.lastFrameTime = currentTime
    },
    
    updatePerformance(library, metrics) {
      this.performance[library] = metrics
    }
  },
  
  beforeUnmount() {
    this.stopStreaming()
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

#app {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  color: #2c3e50;
  background-color: #f5f5f5;
  min-height: 100vh;
}

.header {
  background: #34495e;
  color: white;
  padding: 20px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.header h1 {
  margin-bottom: 15px;
  font-size: 24px;
}

.controls {
  display: flex;
  gap: 15px;
  align-items: center;
  flex-wrap: wrap;
}

button {
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.3s;
  background: #3498db;
  color: white;
}

button:hover {
  background: #2980b9;
}

button.active {
  background: #e74c3c;
}

button.active:hover {
  background: #c0392b;
}

.speed-control {
  display: flex;
  align-items: center;
  gap: 10px;
}

.speed-control select {
  padding: 8px;
  border-radius: 4px;
  border: 1px solid #bdc3c7;
}

.stats {
  display: flex;
  justify-content: space-around;
  background: white;
  padding: 15px;
  margin: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.stat span {
  font-weight: bold;
  font-size: 16px;
  color: #2c3e50;
}

.charts-container {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
  padding: 0 20px;
  margin-bottom: 30px;
}

.chart-wrapper {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  overflow: hidden;
}

.performance-table {
  margin: 20px;
  background: white;
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.performance-table h3 {
  margin-bottom: 15px;
  color: #2c3e50;
}

table {
  width: 100%;
  border-collapse: collapse;
}

th, td {
  padding: 12px;
  text-align: left;
  border-bottom: 1px solid #ecf0f1;
}

th {
  background-color: #34495e;
  color: white;
  font-weight: bold;
}

tr:hover {
  background-color: #f8f9fa;
}

@media (max-width: 768px) {
  .charts-container {
    grid-template-columns: 1fr;
  }
  
  .controls {
    flex-direction: column;
    align-items: flex-start;
  }
  
  .stats {
    flex-direction: column;
    gap: 10px;
  }
}
</style>