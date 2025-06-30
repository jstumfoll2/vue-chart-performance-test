<template>
  <div class="chart-container">
    <h3>µPlot (uplot-vue)</h3>
    <div ref="uplotDiv" class="uplot-chart"></div>
  </div>
</template>

<script>
import uPlot from 'uplot'
import 'uplot/dist/uPlot.min.css'

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
  name: 'UplotChart',
  props: {
    data: {
      type: Object,
      required: true
    }
  },
  data() {
    return {
      uplot: null,
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
    this.performanceMonitor = new PerformanceMonitor('UPlot');
  },
  mounted() {
    this.performanceMonitor.startMonitoring();
    this.initChart();
  },
  beforeUnmount() {
    this.performanceMonitor.stopMonitoring();
    // Export final metrics
    console.log('Final Performance Report:', this.performanceMonitor.exportMetrics());
    
    if (this.uplot) {
      this.uplot.destroy();
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
        const opts = {
          title: '',
          width: this.$refs.uplotDiv.clientWidth,
          height: 350,
          series: [
            {},
            {
              label: 'Chart.js Render Time (ms)',
              stroke: '#e74c3c',
              width: 1
            },
            {
              label: 'Plot.ly Render Time (ms)',
              stroke: '#3498db',
              width: 1
            },
            {
              label: 'Uplot Render Time (ms)',
              stroke: '#2ecc71',
              width: 1
            },
            {
              label: 'WebGl Render Time (ms)',
              stroke: '#f39c12',
              width: 1
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
        };
        
        this.uplot = new uPlot(opts, [], this.$refs.uplotDiv);
        this.plotInitialized = true;
      });
    },
    
    updateChart() {
      if (!this.plotInitialized || !this.data.timestamps.length) return;
      
      this.performanceMonitor.measureOperation('chart_update', () => {
        const timestamps = this.data.timestamps.map(ts => ts.getTime() / 1000);
        const data = [
          timestamps,
          this.data.series1,
          this.data.series2,
          this.data.series3,
          this.data.series4
        ];
        
        this.uplot.setData(data);
      });
      
      // Emit performance data to parent component
      const currentMetrics = this.performanceMonitor.exportMetrics();
      this.$emit('performance', {
        renderTime: currentMetrics.averageRenderTime || 0,
        memoryUsage: currentMetrics.peakMemory || 0,
        // dataPoints: this.data.timestamps.length,
        cpuUsage: this.performanceMonitor.calculateCPUProxy()
      });
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
  /* color: #2c3e50; */
  text-align: center;
}

.uplot-chart {
  height: 350px;
}
</style>