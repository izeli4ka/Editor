<template>
  <div class="filtration-modal">
    <div class="panel-header">
      <button class="close-button" @click="close">
        <label class="button-label">
          <img class="button-image" src="../assets/close.svg" alt="Close" />
        </label>
      </button>
    </div>
    <div class="filtrations-graph">
      <select
        class="filtration-preset"
        @change="applyPreset($event.target.value)"
      >
        <option value="identity" title="Тождественное отображение">
          Identity Transformation
        </option>
        <option value="sharpen" title="Повышение резкости">Sharpen</option>
        <option value="gaussian" title="Фильтр Гаусса">
          Gaussian Filter (3x3)
        </option>
        <option value="boxBlur" title="Прямоугольное размытие">Box Blur</option>
      </select>
      <div class="filtration-content">
        <div class="filtration-inputs">
          <div class="point-inputs">
            <div
              v-for="(row, rowIndex) in kernel"
              class="point-inputs-column"
              :key="rowIndex"
            >
              <div
                v-for="(value, colIndex) in row"
                :key="colIndex"
                class="point-with-prefix"
              >
                <input
                  :value="kernel[rowIndex][colIndex]"
                  @change="(e) => updateKernel(e, rowIndex, colIndex)"
                  :disabled="previewEnabled"
                />
              </div>
            </div>
          </div>
        </div>
        <div class="filtration-actions">
          <div class="custom-checkbox">
            <input type="checkbox" v-model="previewEnabled" />
            <img
              v-if="previewEnabled"
              class="checkbox-image"
              src="../assets/eye.png"
              alt="Lock"
            />
            <img
              v-else
              class="checkbox-image"
              src="../assets/closed-eye.png"
              alt="Unlock"
            />
          </div>
          <button class="filtration-button" @click="applyFiltration">
            Apply
          </button>
          <button class="filtration-button" @click="resetFiltration">
            Reset
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { defineComponent } from "vue";

export default defineComponent({
  name: "FiltrationModal",
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
  data() {
    return {
      previewEnabled: false,
      kernel: [
        [0, 0, 0],
        [0, 1, 0],
        [0, 0, 0],
      ],
      currentImg: null,
      originalImg: null,
    };
  },
  mounted() {
    this.currentImg = this.newImg;
    this.originalImg = this.origImg;
  },
  methods: {
    close() {
      this.$emit("close");
    },
    applyPreset(preset) {
      switch (preset) {
        case "identity":
          this.kernel = [
            [0, 0, 0],
            [0, 1, 0],
            [0, 0, 0],
          ];
          break;
        case "sharpen":
          this.kernel = [
            [0, -1, 0],
            [-1, 5, -1],
            [0, -1, 0],
          ];
          break;
        case "gaussian":
          this.kernel = [
            [1, 2, 1],
            [2, 4, 2],
            [1, 2, 1],
          ];
          break;
        case "boxBlur":
          this.kernel = [
            [1, 1, 1],
            [1, 1, 1],
            [1, 1, 1],
          ];
          break;
        default:
          break;
      }
    },
    updateKernel(event, rowIndex, colIndex) {
      const num = +event.target.value;
      if (!isNaN(num)) {
        this.kernel[rowIndex][colIndex] = num;
      } else {
        this.kernel[rowIndex][colIndex] += 1;
        this.kernel[rowIndex][colIndex] -= 1;
      }
    },
    calculateFiltration() {
      const ctx = this.canvasRef?.getContext("2d");
      const imageData = ctx.getImageData(this.dx, this.dy, this.iw, this.ih);
      const newData = new Uint8ClampedArray(imageData.data.length);

      const paddedData = this.padImageData(
        imageData.data,
        imageData.width,
        imageData.height
      );

      for (let y = 0; y < imageData.height; y++) {
        for (let x = 0; x < imageData.width; x++) {
          for (let c = 0; c < 4; c++) {
            const outputIndex = (y * imageData.width + x) * 4 + c;
            let sum = 0;
            let kernelSum = 0;
            for (let ky = 0; ky < 3; ky++) {
              for (let kx = 0; kx < 3; kx++) {
                const inputIndex =
                  ((y + ky) * (imageData.width + 2) + (x + kx)) * 4 + c;
                sum += paddedData[inputIndex] * this.kernel[ky][kx];
                kernelSum += this.kernel[ky][kx];
              }
            }
            newData[outputIndex] = kernelSum !== 0 ? sum / kernelSum : sum;
          }
        }
      }

      imageData.data.set(newData);
      ctx.putImageData(imageData, this.dx, this.dy);

      this.currentImg = new ImageData(new Uint8ClampedArray(imageData.data), imageData.width, imageData.height);
    },
    padImageData(data, width, height) {
      const paddedWidth = width + 2;
      const paddedHeight = height + 2;
      const paddedData = new Uint8ClampedArray(paddedWidth * paddedHeight * 4);

      for (let y = 0; y < height; y++) {
        for (let x = 0; x < width; x++) {
          const inputIndex = (y * width + x) * 4;
          const outputIndex = ((y + 1) * paddedWidth + x + 1) * 4;
          paddedData.set(
            data.subarray(inputIndex, inputIndex + 4),
            outputIndex
          );
        }
      }

      for (let y = 0; y < paddedHeight; y++) {
        for (let x = 0; x < paddedWidth; x++) {
          const outputIndex = (y * paddedWidth + x) * 4;
          if (
            x === 0 ||
            x === paddedWidth - 1 ||
            y === 0 ||
            y === paddedHeight - 1
          ) {
            const nearestX = Math.max(1, Math.min(x, paddedWidth - 2));
            const nearestY = Math.max(1, Math.min(y, paddedHeight - 2));
            const nearestIndex = (nearestY * paddedWidth + nearestX) * 4;
            paddedData.set(
              paddedData.subarray(nearestIndex, nearestIndex + 4),
              outputIndex
            );
          }
        }
      }
      return paddedData;
    },
    applyFiltration() {
      this.calculateFiltration();
      this.$emit("update:newImg", this.currentImg);
      this.$emit("close");
    },
    resetFiltration() {
      this.previewEnabled = false;
      this.applyPreset("identity");
      const ctx = this.canvasRef?.getContext("2d");
      ctx.putImageData(this.originalImg, this.dx, this.dy);
    },
  },
  watch: {
    previewEnabled(newVal) {
      if (newVal) {
        this.calculateFiltration();
      } else {
        this.resetFiltration();
      }
    },
  },
});
</script>

<style scoped>
.filtration-modal {
  position: absolute;
  top: 0;
  right: 0;
  width: 200px;
  background: #ffffff;
  border: solid 3px #00d9ff;
  display: flex;
  flex-direction: column;
  padding: 10px;
  border-radius: 10px;
}

.filtration-modal .panel-header {
  display: flex;
  justify-content: flex-end;
}

.filtration-modal .panel-header .close-button {
  border: none;
  font-size: 20px;
  padding: 10px;
  cursor: pointer;
  font-weight: bold;
  color: #4f4f4f;
  background: transparent;
}

.filtration-modal .filtrations-graph {
  font-size: small;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.filtration-modal .filtration-content {
  display: flex;
  gap: 10px;
}

.filtration-modal .filtration-content .filtration-inputs {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.filtration-modal .filtration-content .filtration-inputs .point-inputs {
  display: flex;
  gap: 3px;
}

.filtration-modal .filtration-content .filtration-inputs .point-inputs .point-inputs-column {
  display: flex;
  flex-direction: column;
  gap: 3px;
}

.filtration-modal .filtration-content .filtration-inputs .point-inputs .point-with-prefix {
  position: relative;
}

.filtration-modal .filtration-content .filtration-inputs .point-inputs .point-with-prefix input {
  height: 20px;
  width: 28px;
  border-radius: 5px;
  padding-left: 5px;
}

.filtration-modal .filtration-content .filtration-actions {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 5px;
}

.filtration-modal .filtration-content .filtration-actions .filtration-button {
  height: 20px;
  width: 60px;
  background: #00d9ff;
  border-radius: 5px;
  transition: 0.2s;
}

.filtration-modal .filtration-content .filtration-actions .filtration-button:hover {
  background: #a0a0a0;
}

.custom-checkbox {
  position: relative;
  display: inline-block;
  height: 20px;
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

</style>