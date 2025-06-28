<template>
  <div class="chart-container">
    <h3>Chart.js (vue-chartjs)</h3>
    <canvas ref="chartCanvas"></canvas>
  </div>
</template>

<script>
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler
} from 'chart.js'

ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler
)

export default {
  name: 'ChartJsChart',
  props: {
    data: {
      type: Object,
      required: true
    }
  },
  data() {
    return {
      chart: null
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
      const ctx = this.$refs.chartCanvas.getContext('2d')
      
      this.chart = new ChartJS(ctx, {
        type: 'line',
        data: {
          labels: [],
          datasets: [
            {
              label: 'CPU Usage (%)',
              data: [],
              borderColor: '#e74c3c',
              backgroundColor: 'rgba(231, 76, 60, 0.1)',
              tension: 0.1,
              pointRadius: 0
            },
            {
              label: 'Memory Usage (MB)',
              data: [],
              borderColor: '#3498db',
              backgroundColor: 'rgba(52, 152, 219, 0.1)',
              tension: 0.1,
              pointRadius: 0
            },
            {
              label: 'Network I/O (KB/s)',
              data: [],
              borderColor: '#2ecc71',
              backgroundColor: 'rgba(46, 204, 113, 0.1)',
              tension: 0.1,
              pointRadius: 0
            },
            {
              label: 'Disk I/O (KB/s)',
              data: [],
              borderColor: '#f39c12',
              backgroundColor: 'rgba(243, 156, 18, 0.1)',
              tension: 0.1,
              pointRadius: 0
            }
          ]
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          animation: false,
          scales: {
            x: {
              type: 'category',
              ticks: {
                maxTicksLimit: 10
              }
            },
            y: {
              beginAtZero: true
            }
          },
          elements: {
            point: {
              radius: 0
            }
          }
        }
      })
    },
    
    updateChart() {
      if (!this.chart || !this.data.timestamps.length) return
      
      const startTime = performance.now()
      const memoryBefore = performance.memory ? performance.memory.usedJSHeapSize : 0
      
      const labels = this.data.timestamps.map(ts => 
        ts.toLocaleTimeString('en-US', { 
          hour12: false, 
          minute: '2-digit', 
          second: '2-digit' 
        })
      )
      
      this.chart.data.labels = labels
      this.chart.data.datasets[0].data = this.data.series1
      this.chart.data.datasets[1].data = this.data.series2
      this.chart.data.datasets[2].data = this.data.series3
      this.chart.data.datasets[3].data = this.data.series4
      
      this.chart.update('none')
      
      const endTime = performance.now()
      const memoryAfter = performance.memory ? performance.memory.usedJSHeapSize : 0
      
      this.$emit('performance', {
        renderTime: endTime - startTime,
        memoryUsage: (memoryAfter - memoryBefore) / 1024 / 1024,
        cpuUsage: Math.random() * 20 + 10 // Simulated CPU usage
      })
    }
  },
  
  beforeUnmount() {
    if (this.chart) {
      this.chart.destroy()
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

canvas {
  max-height: 350px;
}
</style>