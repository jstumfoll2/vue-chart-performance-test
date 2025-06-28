<template>
  <div class="chart-container">
    <h3>Plotly.js (vue-plotly)</h3>
    <div ref="plotlyDiv" class="plotly-chart"></div>
  </div>
</template>

<script>
import Plotly from 'plotly.js-dist-min'

export default {
  name: 'PlotlyChart',
  props: {
    data: {
      type: Object,
      required: true
    }
  },
  data() {
    return {
      plotInitialized: false
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
      const traces = [
        {
          x: [],
          y: [],
          type: 'scatter',
          mode: 'lines',
          name: 'CPU Usage (%)',
          line: { color: '#e74c3c' }
        },
        {
          x: [],
          y: [],
          type: 'scatter',
          mode: 'lines',
          name: 'Memory Usage (MB)',
          line: { color: '#3498db' }
        },
        {
          x: [],
          y: [],
          type: 'scatter',
          mode: 'lines',
          name: 'Network I/O (KB/s)',
          line: { color: '#2ecc71' }
        },
        {
          x: [],
          y: [],
          type: 'scatter',
          mode: 'lines',
          name: 'Disk I/O (KB/s)',
          line: { color: '#f39c12' }
        }
      ]
      
      const layout = {
        margin: { t: 30, r: 30, b: 50, l: 50 },
        showlegend: true,
        xaxis: { title: 'Time' },
        yaxis: { title: 'Value' },
        height: 350
      }
      
      const config = {
        responsive: true,
        displayModeBar: false
      }
      
      Plotly.newPlot(this.$refs.plotlyDiv, traces, layout, config)
      this.plotInitialized = true
    },
    
    updateChart() {
      if (!this.plotInitialized || !this.data.timestamps.length) return
      
      const startTime = performance.now()
      const memoryBefore = performance.memory ? performance.memory.usedJSHeapSize : 0
      
      const timestamps = this.data.timestamps.map(ts => ts.toISOString())
      
      const update = {
        x: [timestamps, timestamps, timestamps, timestamps],
        y: [this.data.series1, this.data.series2, this.data.series3, this.data.series4]
      }
      
      Plotly.restyle(this.$refs.plotlyDiv, update)
      
      const endTime = performance.now()
      const memoryAfter = performance.memory ? performance.memory.usedJSHeapSize : 0
      
      this.$emit('performance', {
        renderTime: endTime - startTime,
        memoryUsage: (memoryAfter - memoryBefore) / 1024 / 1024,
        cpuUsage: Math.random() * 25 + 15 // Simulated CPU usage
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

.plotly-chart {
  height: 350px;
}
</style>