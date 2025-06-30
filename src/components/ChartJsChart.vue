<template>
  <div class="chart-container">
    <h3>Chart.js (vue-chartjs)</h3>
    <canvas ref="chartCanvas"></canvas>
  </div>
</template>

<script>
import { markRaw } from 'vue'
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  LineController,
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
  LineController,
  Title,
  Tooltip,
  Legend,
  Filler
)

// PerformanceMonitor.js - A utility class for monitoring chart performance
class PerformanceMonitor {
  constructor(chartName) {
    this.chartName = chartName;
    this.metrics = [];
    this.lastCPUTime = performance.now();
    this.frameCount = 0;
    this.isMonitoring = false;
  }

  // Start monitoring performance
  startMonitoring() {
    this.isMonitoring = true;
    this.lastCPUTime = performance.now();
    this.frameCount = 0;
    
    // Monitor frame rate as CPU proxy
    this.frameRateMonitor();
  }

  // Stop monitoring
  stopMonitoring() {
    this.isMonitoring = false;
  }

  // Frame rate monitoring (proxy for CPU usage)
  frameRateMonitor() {
    if (!this.isMonitoring) return;
    
    this.frameCount++;
    requestAnimationFrame(() => this.frameRateMonitor());
  }

  // Measure performance for a specific operation
  measureOperation(operationName, operation) {
    const startTime = performance.now();
    const memoryBefore = this.getMemoryUsage();
    
    // Reset frame counter
    this.frameCount = 0;
    const frameCountStart = this.frameCount;
    
    // Execute the operation
    const result = operation();
    
    // Monitor performance
    const endTime = performance.now();
    const memoryAfter = this.getMemoryUsage();
    const frameCountEnd = this.frameCount;
    
    const metrics = {
      chartType: this.chartName,
      operation: operationName,
      renderTime: endTime - startTime,
      memoryDelta: memoryAfter - memoryBefore,
      memoryUsage: memoryAfter,
      frameRate: frameCountEnd - frameCountStart,
      timestamp: new Date().toISOString()
    };
    
    this.metrics.push(metrics);
    return result;
  }

  // Get memory usage (browser-specific)
  getMemoryUsage() {
    if (performance.memory) {
      return {
        used: performance.memory.usedJSHeapSize / 1024 / 1024, // MB
        total: performance.memory.totalJSHeapSize / 1024 / 1024, // MB
        limit: performance.memory.jsHeapSizeLimit / 1024 / 1024 // MB
      };
    }
    return { used: 0, total: 0, limit: 0 };
  }

  // Calculate CPU usage proxy based on frame timing
  calculateCPUProxy() {
    const now = performance.now();
    const timeDelta = now - this.lastCPUTime;
    const expectedFrames = timeDelta / 16.67;
    const actualFrames = this.frameCount;
        
    // CPU proxy: higher values indicate more CPU stress
    const cpuProxy = Math.max(0, (expectedFrames - actualFrames) / expectedFrames * 100);
    
    this.lastCPUTime = now;
    this.frameCount = 0;
    
    return cpuProxy;
  }

  // Log metrics
  logMetrics(metrics) {
    console.log(`[${metrics.chartType}] ${metrics.operation}:`, {
      'Render Time': `${metrics.renderTime.toFixed(2)}ms`,
      'Memory Used': `${metrics.memoryUsage.used.toFixed(2)}MB`,
      'Memory Delta': `${metrics.memoryDelta.used?.toFixed(2) || 0}MB`,
      'Frame Rate Proxy': `${metrics.frameRate} frames`,
      'CPU Proxy': `${this.calculateCPUProxy().toFixed(2)}%`
    });
  }

  // Export metrics for analysis
  exportMetrics() {
    return {
      chartType: this.chartName,
      totalMetrics: this.metrics.length,
      averageRenderTime: this.metrics.reduce((sum, m) => sum + m.renderTime, 0) / this.metrics.length,
      peakMemory: Math.max(...this.metrics.map(m => m.memoryUsage.used)),
      metrics: this.metrics
    };
  }
}

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
      chart: null,
      plotInitialized: false,
      performanceMonitor: null,
      performanceStats: {
        renderTimes: [],
        memoryUsages: [],
        frameRates: []
      }
    }
  },
  created() {
    this.performanceMonitor = new PerformanceMonitor('Chart.js');
  },
  mounted() {
    this.performanceMonitor.startMonitoring();
    this.initChart();
  },
  beforeUnmount() {
    this.performanceMonitor.stopMonitoring();
    // Export final metrics
    console.log('Final Performance Report:', this.performanceMonitor.exportMetrics());
    
    if (this.chart) {
      this.chart.destroy();
    }
  },
  watch: {
    data: {
      handler() {
        this.updateChart();
      },
      deep: true
    }
  },
  methods: {
    initChart() {
      this.performanceMonitor.measureOperation('chart_init', () => {
        const ctx = this.$refs.chartCanvas.getContext('2d');
        
        // Use markRaw to prevent Vue from making Chart.js instance reactive
        this.chart = markRaw(new ChartJS(ctx, {
          type: 'line',
          options: {
            responsive: true,
            maintainAspectRatio: false,
            animation: false,
            interaction: {
              intersect: false
            },
            scales: {
              x: {
                type: 'category',
                ticks: {
                  maxTicksLimit: 10
                },
                display: true
              },
              y: {
                beginAtZero: true,
                display: true
              }
            },
            elements: {
              point: {
                radius: 0
              },
              line: {
                borderWidth: 2
              }
            },
            plugins: {
              legend: {
                display: true,
                position: 'bottom'
              }
            }
          },
          data: {
            labels: [],
            datasets: [
              {
                label: 'Chart.js Render Time (ms)',
                data: [],
                borderColor: '#e74c3c',
                backgroundColor: 'rgba(231, 76, 60, 0.1)',
                tension: 0.1,
                pointRadius: 0
              },
              {
                label: 'Plot.ly Render Time (ms)',
                data: [],
                borderColor: '#3498db',
                backgroundColor: 'rgba(52, 152, 219, 0.1)',
                tension: 0.1,
                pointRadius: 0
              },
              {
                label: 'Uplot Render Time (ms)',
                data: [],
                borderColor: '#2ecc71',
                backgroundColor: 'rgba(46, 204, 113, 0.1)',
                tension: 0.1,
                pointRadius: 0
              },
              {
                label: 'WebGl Render Time (ms)',
                data: [],
                borderColor: '#f39c12',
                backgroundColor: 'rgba(243, 156, 18, 0.1)',
                tension: 0.1,
                pointRadius: 0
              }
            ]
          }
        }));
        
        this.plotInitialized = true;
      });
    },
    
    updateChart() {
      if (!this.plotInitialized || !this.data || !this.data.timestamps || !this.data.timestamps.length) return;
      
      try {
        this.performanceMonitor.measureOperation('chart_update', () => {
          // Create non-reactive copies of the data
          const labels = this.data.timestamps.map(ts => 
            ts.toLocaleTimeString('en-US', { 
              hour12: false, 
              minute: '2-digit', 
              second: '2-digit' 
            })
          );
          
          const series1Data = [...(this.data.series1 || [])];
          const series2Data = [...(this.data.series2 || [])];
          const series3Data = [...(this.data.series3 || [])];
          const series4Data = [...(this.data.series4 || [])];

          // Update chart data with non-reactive copies
          this.chart.data.labels = labels;
          this.chart.data.datasets[0].data = series1Data;
          this.chart.data.datasets[1].data = series2Data;
          this.chart.data.datasets[2].data = series3Data;
          this.chart.data.datasets[3].data = series4Data;

          // Update the chart with no animation for better performance
          this.chart.update('none');
        });
        
        // Emit performance data to parent component
        const currentMetrics = this.performanceMonitor.exportMetrics();
        this.$emit('performance', {
          renderTime: currentMetrics.averageRenderTime || 0,
          memoryUsage: currentMetrics.peakMemory || 0,
          // dataPoints: this.data.timestamps.length,
          cpuUsage: this.performanceMonitor.calculateCPUProxy()
        });
      } catch (error) {
        console.error('Chart.js update error:', error);
      }
    }
  }
}
</script>

<style scoped>
.chart-container {
  padding: 20px;
  height: 500px;
}

.chart-container h3 {
  margin-bottom: 15px;
  color: #2c3e50;
  text-align: center;
}

canvas {
  max-height: 550px;
  width: 100% !important;
  height: 100% !important;
}
</style>