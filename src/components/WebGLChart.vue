<template>
  <div class="chart-container">
    <h3>WebGL Plot</h3>
    <div class="chart-wrapper">
      <canvas ref="webglCanvas" class="webgl-canvas"></canvas>
      <div class="axis-labels">
        <div class="y-axis">
          <div class="y-label y-max">{{ formatValue(yMax) }}</div>
          <div class="y-label y-mid">{{ formatValue((yMax + yMin) / 2) }}</div>
          <div class="y-label y-min">{{ formatValue(yMin) }}</div>
        </div>
        <div class="x-axis">
          <div class="x-label x-start">{{ formatTime(xMin) }}</div>
          <div class="x-label x-mid">{{ formatTime(xMid) }}</div>
          <div class="x-label x-end">{{ formatTime(xMax) }}</div>
        </div>
      </div>
      <div class="grid-overlay" ref="gridOverlay"></div>
    </div>
    <div class="legend">
      <div class="legend-item">
        <span class="legend-color" style="background-color: #e74c3c;"></span>
        <span>Chart.js Render Time (ms)</span>
      </div>
      <div class="legend-item">
        <span class="legend-color" style="background-color: #3498db;"></span>
        <span>Plot.ly Render Time (ms)</span>
      </div>
      <div class="legend-item">
        <span class="legend-color" style="background-color: #2ecc71;"></span>
        <span>Uplot Render Time (ms)</span>
      </div>
      <div class="legend-item">
        <span class="legend-color" style="background-color: #f39c12;"></span>
        <span>WebGl Render Time (ms)</span>
      </div>
    </div>
  </div>
</template>

<script>
import { WebglPlot, WebglLine, ColorRGBA } from 'webgl-plot'

// Import the PerformanceMonitor class from the previous artifact
class PerformanceMonitor {
  constructor(chartName) {
    this.chartName = chartName;
    this.metrics = [];
    this.lastCPUTime = performance.now();
    this.frameCount = 0;
    this.isMonitoring = false;
  }

  startMonitoring() {
    this.isMonitoring = true;
    this.lastCPUTime = performance.now();
    this.frameCount = 0;
    this.frameRateMonitor();
  }

  stopMonitoring() {
    this.isMonitoring = false;
  }

  frameRateMonitor() {
    if (!this.isMonitoring) return;
    this.frameCount++;
    requestAnimationFrame(() => this.frameRateMonitor());
  }

  measureOperation(operationName, operation) {
    const startTime = performance.now();
    const memoryBefore = this.getMemoryUsage();
    const frameCountStart = this.frameCount;
    
    const result = operation();
    
    const endTime = performance.now();
    const memoryAfter = this.getMemoryUsage();
    const frameCountEnd = this.frameCount;
    
    const metrics = {
    chartType: this.chartName,
    operation: operationName,
    renderTime: endTime - startTime,
    memoryDelta: memoryAfter.used - memoryBefore.used,
    memoryUsage: memoryAfter,
    frameRate: frameCountEnd - frameCountStart,
    timestamp: new Date().toISOString()
    };
    
    this.metrics.push(metrics);

    return result;
  }

  getMemoryUsage() {
    if (performance.memory) {
      return {
        used: performance.memory.usedJSHeapSize / 1024 / 1024,
        total: performance.memory.totalJSHeapSize / 1024 / 1024,
        limit: performance.memory.jsHeapSizeLimit / 1024 / 1024
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
      maxPoints: 1000,
      performanceMonitor: null,
      yMin: 0,
      yMax: 100,
      xMin: null,
      xMid: null,
      xMax: null
    }
  },
  created() {
    this.performanceMonitor = new PerformanceMonitor('WebGL');
  },
  mounted() {
    this.performanceMonitor.startMonitoring();
    this.initChart();
    this.drawGrid();
    // Handle canvas resize
    window.addEventListener('resize', this.handleResize);
  },
  beforeUnmount() {
    this.performanceMonitor.stopMonitoring();
    console.log('WebGL Final Performance Report:', this.performanceMonitor.exportMetrics());
    window.removeEventListener('resize', this.handleResize);
    
    // Clean up WebGL resources
    if (this.webglPlot) {
      this.webglPlot = null;
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
        const canvas = this.$refs.webglCanvas;
        const rect = canvas.getBoundingClientRect();
        canvas.width = rect.width;
        canvas.height = 300;
        
        this.webglPlot = new WebglPlot(canvas);
        
        const colors = [
          new ColorRGBA(0.9, 0.3, 0.23, 1), // #e74c3c
          new ColorRGBA(0.2, 0.6, 0.86, 1), // #3498db
          new ColorRGBA(0.18, 0.8, 0.44, 1), // #2ecc71
          new ColorRGBA(0.95, 0.61, 0.07, 1) // #f39c12
        ];
        
        for (let i = 0; i < 4; i++) {
          const line = new WebglLine(colors[i], this.maxPoints);
          this.lines.push(line);
          this.webglPlot.addLine(line);
        }
      });
    },
    
    updateChart() {
      if (!this.webglPlot || !this.data.timestamps.length) return;
      
      this.performanceMonitor.measureOperation('chart_update', () => {
        const dataLength = this.data.timestamps.length;
        const series = [this.data.series1, this.data.series2, this.data.series3, this.data.series4];
        
        // Calculate global min/max for consistent scaling across all series
        const allValues = series.flat();
        this.yMax = Math.max(...allValues);
        this.yMin = Math.min(...allValues);
        const range = this.yMax - this.yMin || 1;
        
        // Update time axis labels
        this.xMin = this.data.timestamps[0];
        this.xMax = this.data.timestamps[dataLength - 1];
        this.xMid = new Date((this.xMin.getTime() + this.xMax.getTime()) / 2);
        
        for (let i = 0; i < 4; i++) {
          const line = this.lines[i];
          
          // Limit data points to prevent buffer overflow
          const pointsToShow = Math.min(dataLength, this.maxPoints);
          const startIndex = Math.max(0, dataLength - pointsToShow);
          
          // Clear the entire xy array to prevent ghost points
          for (let k = 0; k < line.xy.length; k++) {
            line.xy[k] = 0;
          }
          
          // Set the number of points for this line
          line.numPoints = pointsToShow;
          
          for (let j = 0; j < pointsToShow; j++) {
            const dataIndex = startIndex + j;
            
            // Normalize X coordinate to [-1, 1] based on time progression
            const x = (j / Math.max(pointsToShow - 1, 1)) * 2 - 1;
            
            // Normalize Y coordinate to [-1, 1] based on value range
            const y = ((series[i][dataIndex] - this.yMin) / range) * 2 - 1;
            
            // Directly set coordinates in the line's data array
            line.xy[2 * j] = x;      // X coordinate
            line.xy[2 * j + 1] = y;  // Y coordinate
          }
          
          // Force buffer update
          line.webglNumPoints = pointsToShow;
        }
        
        // Render the plot
        this.webglPlot.update();
        
        // Update grid after data changes
        this.$nextTick(() => {
          this.drawGrid();
        });
      });
      
      // Emit performance data to parent component
      const currentMetrics = this.performanceMonitor.exportMetrics();
      this.$emit('performance', {
        renderTime: currentMetrics.averageRenderTime || 0,
        memoryUsage: currentMetrics.peakMemory || 0,
        // dataPoints: this.data.timestamps.length,
        cpuUsage: this.performanceMonitor.calculateCPUProxy()
      });
    },
    
    drawGrid() {
      const gridOverlay = this.$refs.gridOverlay;
      if (!gridOverlay) return;
      
      // Clear existing grid
      gridOverlay.innerHTML = '';
      
      // Create grid lines
      const gridSvg = document.createElementNS('http://www.w3.org/2000/svg', 'svg');
      gridSvg.setAttribute('width', '100%');
      gridSvg.setAttribute('height', '300px');
      gridSvg.style.position = 'absolute';
      gridSvg.style.top = '0';
      gridSvg.style.left = '0';
      gridSvg.style.pointerEvents = 'none';
      
      // Horizontal grid lines (5 lines)
      for (let i = 0; i <= 4; i++) {
        const y = (i / 4) * 300;
        const line = document.createElementNS('http://www.w3.org/2000/svg', 'line');
        line.setAttribute('x1', '0');
        line.setAttribute('y1', y);
        line.setAttribute('x2', '100%');
        line.setAttribute('y2', y);
        line.setAttribute('stroke', '#e0e0e0');
        line.setAttribute('stroke-width', '1');
        line.setAttribute('stroke-dasharray', '2,2');
        gridSvg.appendChild(line);
      }
      
      // Vertical grid lines (5 lines)
      for (let i = 0; i <= 4; i++) {
        const x = (i / 4) * 100 + '%';
        const line = document.createElementNS('http://www.w3.org/2000/svg', 'line');
        line.setAttribute('x1', x);
        line.setAttribute('y1', '0');
        line.setAttribute('x2', x);
        line.setAttribute('y2', '300');
        line.setAttribute('stroke', '#e0e0e0');
        line.setAttribute('stroke-width', '1');
        line.setAttribute('stroke-dasharray', '2,2');
        gridSvg.appendChild(line);
      }
      
      gridOverlay.appendChild(gridSvg);
    },
    
    handleResize() {
      if (this.webglPlot) {
        const canvas = this.$refs.webglCanvas;
        const rect = canvas.getBoundingClientRect();
        canvas.width = rect.width;
        canvas.height = 300;
        this.drawGrid();
        this.updateChart();
      }
    },
    
    formatValue(value) {
      if (value === null || value === undefined) return '0';
      return value.toFixed(1);
    },
    
    formatTime(date) {
      if (!date) return '';
      return date.toLocaleTimeString('en-US', { 
        hour12: false, 
        hour: '2-digit', 
        minute: '2-digit', 
        second: '2-digit' 
      });
    }
  }
}
</script>

<style scoped>
.chart-container {
  padding: 20px;
  height: 450px;
}

.chart-container h3 {
  margin-bottom: 15px;
  color: #2c3e50;
  text-align: center;
}

.chart-wrapper {
  position: relative;
  width: calc(100% - 60px);
  height: 340px;
  margin-left: 60px;
  margin-bottom: 40px;
}

.webgl-canvas {
  width: 100%;
  height: 300px;
  border: 1px solid #ddd;
  border-radius: 4px;
  display: block;
}

.grid-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 300px;
  pointer-events: none;
  z-index: 1;
}

.axis-labels {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 2;
}

.y-axis {
  position: absolute;
  left: -55px;
  top: 0;
  height: 300px;
  width: 50px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  align-items: flex-end;
  padding: 5px 0;
}

.y-label {
  font-size: 11px;
  color: #666;
  background: white;
  padding: 2px 4px;
  border-radius: 2px;
  border: 1px solid #ddd;
  white-space: nowrap;
}

.x-axis {
  position: absolute;
  bottom: -35px;
  left: 0;
  width: 100%;
  height: 30px;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  padding: 5px;
}

.x-label {
  font-size: 11px;
  color: #666;
  background: white;
  padding: 2px 4px;
  border-radius: 2px;
  border: 1px solid #ddd;
  white-space: nowrap;
  transform: translateX(-50%);
}

.x-label.x-start {
  transform: translateX(0);
}

.x-label.x-end {
  transform: translateX(-100%);
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