<template>
  <div class="correction-info-panel">
    <div class="panel-header">
      <button class="close-button" @click="close">
        <label class="button-label">
          <img class="button-image" src="../assets/close.svg" alt="Close" />
        </label>
      </button>
    </div>
    <div class="correction-inputs">
      <div class="point-inputs">
        <div class="point-with-prefix">
          <label for="x1" :style="{ left: '3px' }">x1</label>
          <input id="x1" :value="point1x" @change="updateX1" :disabled="previewEnabled" />
        </div>
        <div class="point-with-prefix">
          <label for="y1" :style="{ left: '3px' }">y1</label>
          <input id="y1" :value="point1y" @change="updateY1" :disabled="previewEnabled" />
        </div>
      </div>
      <div class="point-inputs">
        <div class="point-with-prefix">
          <label for="x2" :style="{ left: '3px' }">x2</label>
          <input id="x2" :value="point2x" @change="updateX2" :disabled="previewEnabled" />
        </div>
        <div class="point-with-prefix">
          <label for="y2" :style="{ left: '3px' }">y2</label>
          <input id="y2" :value="point2y" @change="updateY2" :disabled="previewEnabled" />
        </div>
      </div>
      <div class="correction-preview">
        <div class="custom-checkbox">
          <input type="checkbox" v-model="previewEnabled" />
          <img v-if="previewEnabled" class="checkbox-image" src="../assets/eye.png" alt="Lock" />
          <img v-else class="checkbox-image" src="../assets/closed-eye.png" alt="Unlock" />
        </div>
      </div>
    </div>
    <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
    <div class="correction-actions">
      <button class="correction-button" @click="applyCorrection" :disabled="imgCorrected">Apply</button>
      <button class="correction-button" @click="resetValues">Reset</button>
    </div>
    <div class="correction-graph">
      <canvas id="chart" width="100" height="100" ref="chart"></canvas>
    </div>
  </div>
</template>

<script>
import { defineComponent } from "vue";
import Chart from "chart.js/auto";

export default defineComponent({
  name: "CorrectionModal",
  props: {
    state: String,
    canvasRef: Object,
    origImg: Object,
    newImg: Object,
    dx: Number,
    dy: Number,
    ih: Number,
    iw: Number,
  },
  mounted() {
    this.chartRef = this.$refs.chart;
  },
  data() {
    return {
      point1x: 0,
      point1y: 0,
      point2x: 255,
      point2y: 255,
      previewEnabled: false,
      chartInstance: null,
      imgCorrected: false,
      errorMessage: '',
    };
  },
  methods: {
    close() {
      this.$emit("close");
    },
    normalizeHistogram(histogram) {
      const max = Math.max(...histogram);
      return histogram.map((value) => (value / max) * 255);
    },
    buildGraph() {
      const img = new Image();
      img.onload = () => {
        const ctx = this.canvasRef?.getContext("2d");
        try {
          const imageData = ctx.getImageData(this.dx, this.dy, this.iw, this.ih);
          const data = imageData.data;
          const redHistogram = new Array(256).fill(0);
          const greenHistogram = new Array(256).fill(0);
          const blueHistogram = new Array(256).fill(0);

          for (let i = 0; i < data.length; i += 4) {
            const red = data[i];
            const green = data[i + 1];
            const blue = data[i + 2];
            redHistogram[red]++;
            greenHistogram[green]++;
            blueHistogram[blue]++;
          }
          const normalizedRedHistogram = this.normalizeHistogram(redHistogram);
          const normalizedGreenHistogram = this.normalizeHistogram(greenHistogram);
                    const normalizedBlueHistogram = this.normalizeHistogram(blueHistogram);
          this.drawHistograms(normalizedRedHistogram, normalizedGreenHistogram, normalizedBlueHistogram);
        } catch (e) {
          console.log(e);
        }
      };
      img.src = this.origImg.src;
    },
    drawHistograms(redHistogram, greenHistogram, blueHistogram) {
      const ctx = this.chartRef?.getContext("2d");

      if (this.chartInstance) {
        this.chartInstance.destroy();
      }
      const canvasSize = 600;
      this.chartRef.width = canvasSize;
      this.chartRef.height = canvasSize;

      this.chartInstance = new Chart(ctx, {
        type: "scatter",
        data: {
          labels: Array.from({ length: 256 }, (_, i) => i),
          datasets: [
            {
              label: "Red",
              data: redHistogram.map((y, x) => ({ x, y })),
              backgroundColor: "rgba(255, 0, 0, 0.8)",
              borderColor: "rgba(255, 0, 0, 0.8)",
              showLine: false,
              pointStyle: "circle",
              pointRadius: 3,
            },
            {
              label: "Green",
              data: greenHistogram.map((y, x) => ({ x, y })),
              backgroundColor: "rgba(0, 255, 0, 0.8)",
              borderColor: "rgba(0, 255, 0, 0.8)",
              showLine: false,
              pointStyle: "circle",
              pointRadius: 3,
            },
            {
              label: "Blue",
              data: blueHistogram.map((y, x) => ({ x, y })),
              backgroundColor: "rgba(0, 0, 255, 0.8)",
              borderColor: "rgba(0, 0, 255, 0.8)",
              showLine: false,
              pointStyle: "circle",
              pointRadius: 3,
            },
            {
              label: "line",
              data: [
                { x: 0, y: this.point1y },
                { x: this.point1x, y: this.point1y },
                { x: this.point2x, y: this.point2y },
                { x: 255, y: this.point2y },
              ],
              borderColor: "rgba(0,0,0,1)",
              borderWidth: 1,
              fill: false,
              showLine: true,
            },
          ],
        },
        options: {
          animation: false,
          aspectRatio: 1,
          scales: {
            x: {
              type: "linear",
              position: "bottom",
              min: 0,
              max: 260,
              ticks: {
                stepSize: 20,
              },
            },
            y: {
              beginAtZero: true,
              min: 0,
              max: 260,
              ticks: {
                stepSize: 15,
              },
            },
          },
        },
      });
    },
    calculateCorrection() {
      try {
        const lut = [];
        for (let i = 0; i < this.point1x; i++) {
          lut[i] = this.point1y;
        }
        for (let i = this.point1x; i < this.point2x; i++) {
          const slope = (this.point2y - this.point1y) / (this.point2x - this.point1x);
          let correctedValue = this.point1y + slope * (i - this.point1x);
          correctedValue = Math.max(0, Math.min(255, correctedValue));
          lut[i] = correctedValue;
        }
        for (let i = this.point2x; i < 256; i++) {
          lut[i] = this.point2y;
        }

        const ctx = this.canvasRef?.getContext("2d");
        const imageData = ctx.getImageData(this.dx, this.dy, this.iw, this.ih);
        const data = imageData.data;
        for (let i = 0; i < data.length; i += 4) {
          data[i] = lut[data[i]];
          data[i + 1] = lut[data[i + 1]];
          data[i + 2] = lut[data[i + 2]];
        }

        ctx.putImageData(imageData, this.dx, this.dy);
        this.$emit("updateNewImgData", data);
        this.imgCorrected = true;
      } catch (e) {
        console.log(e);
      }
    },
    revertCorrection() {
      this.$emit("revertNewImg");
      this.imgCorrected = false;
    },
        applyCorrection() {
      this.$emit("close");
      this.calculateCorrection();
      this.buildGraph();
    },
    resetValues() {
      this.point1x = 0;
      this.point1y = 0;
      this.point2x = 255;
      this.point2y = 255;
      this.revertCorrection();
    },
    updateX1(event) {
      const num = +event.target.value;
      this.errorMessage = '';

      if (isNaN(num) || num < 0 || num >= this.point2x) {
        this.errorMessage = 'x1 должен быть меньше, чем x2 и не меньше 0.';
        return;
      }

      this.point1x = num;
      this.buildGraph();
    },
    updateX2(event) {
      const num = +event.target.value;
      this.errorMessage = '';

      if (isNaN(num) || num > 255 || num <= this.point1x) {
        this.errorMessage = 'x2 должен быть больше, чем x1 и не больше 255.';
        return;
      }

      this.point2x = num;
      this.buildGraph();
    },
    updateY1(event) {
      const num = +event.target.value;

      if (!isNaN(num) && num >= 0 && num < 255) {
        this.point1y = num;
        this.buildGraph();
      } else {
        this.point1y += 1;
        this.point1y -= 1;
      }
    },
    updateY2(event) {
      const num = +event.target.value;

      if (!isNaN(num) && num <= 255 && 0 <= num) {
        this.point2y = num;
        this.buildGraph();
      } else {
        this.point2y += 1;
        this.point2y -= 1;
      }
    },
  },
  watch: {
    newImg() {
      this.buildGraph();
    },
    previewEnabled(newVal) {
      if (newVal) {
        this.calculateCorrection();
      } else {
        this.revertCorrection();
      }
    },
  },
});
</script>

<style scoped>
.correction-info-panel {
  position: absolute;
  top: 0;
  right: 0;
  width: 200px;
  background: #ffffff;
  border: solid 3px #00d9ff;
  display: flex;
  flex-direction: column;
  gap: 5px;
  padding: 5px 10px;
  border-radius: 10px;
}

.correction-info-panel .panel-header {
  display: flex;
  justify-content: end;
}

.correction-info-panel .panel-header .close-button {
  border: none;
  font-size: 20px;
  padding: 10px;
  cursor: pointer;
  font-weight: bold;
  color: #4f4f4f;
  background: transparent;
}

.correction-info-panel .correction-graph {
  font-size: small;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.correction-actions {
  display: flex;
  justify-content: center;
  gap: 5px;
}

.correction-actions .correction-button {
  height: 20px;
  width: 60px;
  background: #00d9ff;
  border-radius: 5px;
  transition: 0.2s;
}

.correction-actions .correction-button:hover {
  background: #a0a0a0;
}

.correction-inputs {
  display: flex;
  flex-direction: column;
  gap: 5px;
  align-items: center;
}

.correction-inputs .point-inputs {
  display: flex;
  gap: 5px;
}

.correction-inputs .point-inputs .point-with-prefix {
  position: relative;
}

.correction-inputs .point-inputs .point-with-prefix input {
  height: 20px;
  width: 25px;
  border-radius: 5px;
  padding-left: 20px;
  padding-right: 20px;
}

.correction-inputs .point-inputs .point-with-prefix label {
  color: #848484;
  position: absolute;
  top: 9px;
  transform: translateY(-50%);
}

.custom-checkbox {
  position: relative;
  display: inline-block;
}

.custom-checkbox input,
.custom-checkbox .checkbox-image {
  width: 20px;
  height: 20px;
  cursor: pointer;
}

.custom-checkbox input[type="checkbox"] {
  position: absolute;
  opacity: 0;
  cursor: pointer;
}

.custom-checkbox input[type="checkbox"]:hover {
  border-color: #a0a0a0;
}

.error-message {
  color: red;
  font-size: 12px;
}

.button-label {
  cursor: pointer;
}

.button-image {
  width: 20px;
  height: 20px;
}
</style>
