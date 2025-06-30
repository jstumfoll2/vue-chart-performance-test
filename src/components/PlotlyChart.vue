<template>
  <div class="chart-container">
    <h3>Plotly.js (vue-plotly)</h3>
    <div ref="plotlyDiv" class="plotly-chart"></div>
  </div>
</template>

<script>
import Plotly from 'plotly.js-dist-min';

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
  name: 'PlotlyChart',
  props: {
    data: {
      type: Object,
      required: true
    }
  },
  data() {
    return {
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
    this.performanceMonitor = new PerformanceMonitor('Plotly');
  },
  mounted() {
    this.performanceMonitor.startMonitoring();
    this.initChart();
  },
  beforeUnmount() {
    this.performanceMonitor.stopMonitoring();
    // Export final metrics
    console.log('Final Performance Report:', this.performanceMonitor.exportMetrics());
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
        const traces = [
          {
            x: [],
            y: [],
            type: 'scatter',
            mode: 'lines',
            name: 'Chart.js Render Time (ms)',
            line: { color: '#e74c3c' }
          },
          {
            x: [],
            y: [],
            type: 'scatter',
            mode: 'lines',
            name: 'Plot.ly Render Time (ms)',
            line: { color: '#3498db' }
          },
          {
            x: [],
            y: [],
            type: 'scatter',
            mode: 'lines',
            name: 'Uplot Render Time (ms)',
            line: { color: '#2ecc71' }
          },
          {
            x: [],
            y: [],
            type: 'scatter',
            mode: 'lines',
            name: 'WebGl Render Time (ms)',
            line: { color: '#f39c12' }
          }
        ];
        
        const layout = {
          margin: { t: 30, r: 30, b: 50, l: 50 },
          showlegend: true,
          legend: {
            y: -0.75,
            x: -0.25,
            yanchor: "bottom",
          },
          xaxis: { title: 'Time' },
          yaxis: { title: 'Value' },
          height: 500
        };
        
        const config = {
          responsive: true,
          displayModeBar: false
        };
        
        Plotly.newPlot(this.$refs.plotlyDiv, traces, layout, config);
        this.plotInitialized = true;
      });
    },
    
    updateChart() {
      if (!this.plotInitialized || !this.data.timestamps.length) return;
      
      this.performanceMonitor.measureOperation('chart_update', () => {
        const timestamps = this.data.timestamps.map(ts => ts.toISOString());
        
        const update = {
          x: [timestamps, timestamps, timestamps, timestamps],
          y: [this.data.series1, this.data.series2, this.data.series3, this.data.series4]
        };
        
        Plotly.restyle(this.$refs.plotlyDiv, update);
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

.plotly-chart {
  height: 350px;
}
</style>