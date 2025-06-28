<template>
  <div class="chart-container">
    <h3>µPlot (uplot-vue)</h3>
    <div ref="uplotDiv" class="uplot-chart"></div>
  </div>
</template>

<script>
import uPlot from 'uplot'
import 'uplot/dist/uPlot.min.css'

export default {
  name: 'UplotChart',
  props: {
    data: {
      type: Object,
      required: true
    }
  },
  data() {
    return {
      uplot: null
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
      const opts = {
        title: '',
        width: this.$refs.uplotDiv.clientWidth,
        height: 350,
        series: [
          {},
          {
            label: 'CPU Usage (%)',
            stroke: '#e74c3c',
            width: 2
          },
          {
            label: 'Memory Usage (MB)',
            stroke: '#3498db',
            width: 2
          },
          {
            label: 'Network I/O (KB/s)',
            stroke: '#2ecc71',
            width: 2
          },
          {
            label: 'Disk I/O (KB/s)',
            stroke: '#f39c12',
            width: 2
          }
        ],
        axes: [
          {
            values: (self, ticks) => ticks.map(rawValue => 
              new Date(rawValue * 1000).toLocaleTimeString('en-US', {
                hour12: false,
                minute: '2-digit',
                second: '2-digit'
              })
            )
          },
          {
            label: 'Value'
          }
        ]
      }
      
      this.uplot = new uPlot(opts, [], this.$refs.uplotDiv)
    },
    
    updateChart() {
      if (!this.uplot || !this.data.timestamps.length) return
      
      const startTime = performance.now()
      const memoryBefore = performance.memory ? performance.memory.usedJSHeapSize : 0
      
      const timestamps = this.data.timestamps.map(ts => ts.getTime() / 1000)
      const data = [
        timestamps,
        this.data.series1,
        this.data.series2,
        this.data.series3,
        this.data.series4
      ]
      
      this.uplot.setData(data)
      
      const endTime = performance.now()
      const memoryAfter = performance.memory ? performance.memory.usedJSHeapSize : 0
      
      this.$emit('performance', {
        renderTime: endTime - startTime,
        memoryUsage: (memoryAfter - memoryBefore) / 1024 / 1024,
        cpuUsage: Math.random() * 15 + 5 // Simulated CPU usage
      })
    }
  },
  
  beforeUnmount() {
    if (this.uplot) {
      this.uplot.destroy()
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

.uplot-chart {
  height: 350px;
}
</style>