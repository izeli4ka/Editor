<template>
  <div class="editor">
    <MainCanvas
      ref="canvas"
      :originalWidth="originalWidth"
      :originalHeight="originalHeight"
      :state="state"
      :img="img"
      :origImg="origImg"
      :newImg="newImg"
      :scale="scale"
      :interpolationCb="interpolationCb"
      :newiw="newiw"
      :newih="newih"
      :isShowCorrection="isShowCorrection"
      :isShowFiltration="isShowFiltration"
      @updateImageSizes="updateImageSizes"
      @updateColor="updateColor"
      @updateCoordinates="updateCoordinates"
      @closeCorrection="closeCorrection"
      @closeFiltration="closeFiltration"
      @revertNewImg="revertNewImg"
      @updateNewImgData="updateNewImgData"
    />
    <StatusBar
      :state="state"
      :imageHeight="ih"
      :imageWidth="iw"
      :pickedColor="pickedColor"
      :xMouse="xMouse"
      :yMouse="yMouse"
      :scale="scale"
      @updateScale="(value) => (scale = value)"
    />
  </div>
  <ModalWindow v-show="isShowModal" @close="closeModal">
  <template v-slot:body>
    <div class="modal">
      <select
        class="resize-unit"
        id="resizeUnit"
        v-model="resizeUnit"
        @change="updateModal($event)"
      >
        <option value="percentage">Percentage</option>
        <option value="pixels">Pixels</option>
      </select>
      <div class="sizes">
        <div class="input-with-prefix">
          <label for="width" :style="{ left: '3px' }">Width</label>
          <label for="width" class="type" :style="{ right: '3px' }">
            {{ resizeUnit === "pixels" ? "px" : "%" }}
          </label>
          <input id="width" :value="Math.floor(newiw)" @change="updateNewiw" />
        </div>
        <div class="custom-checkbox">
          <input type="checkbox" v-model="bind" />
          <img
            v-if="bind"
            class="checkbox-image"
            src="../assets/lock.png"
            alt="Заблокировать"
          />
          <img
            v-else
            class="checkbox-image"
            src="../assets/unlock.png"
            alt="Разблокировать"
          />
        </div>
        <div class="input-with-prefix">
          <label for="height" :style="{ left: '6px' }">Height</label>
          <label for="height" class="type" :style="{ right: '3px' }">
            {{ resizeUnit === "pixels" ? "px" : "%" }}
          </label>
          <input id="height" :value="Math.floor(newih)" @change="updateNewih" />
        </div>
      </div>
      <div class="interpolation">
        <label for="interpolation">Интерполяция:</label>
        <select v-model="interpolation" id="interpolation">
          <option></option>
          <option
            value="nearestNeighbor"
            title="Ближайший сосед: Каждому пикселю в новом изображении присваивается значение ближайшего пикселя в исходном изображении."
          >
            Ближайший сосед
          </option>
        </select>
      </div>
      <div class="sizes-counter">
        <p>Сейчас: {{ getResolutionNow() }} МП</p>
        <p>После: {{ getResolutionAfter() }} МП</p>
      </div>
    </div>
  </template>
  <template v-slot:footer>
    <button @click="handleConfirm" class="confirm-button">Подтвердить</button>
  </template>
</ModalWindow>
  <SideBar
    v-show="isShowPanel"
    :state="state"
    :pickedColor="pickedColor"
    :xMouse="xMouse"
    :yMouse="yMouse"
    @close="closePanel"
    @changeState="(value) => $emit('changeState', value)"
  />
</template>

<script>
import { defineComponent } from "vue";

import ModalWindow from "./ModalWindow";
import StatusBar from "./StatusBar";
import MainCanvas from "./MainCanvas.vue";
import SideBar from "./SideBar.vue";

export default defineComponent({
  name: "MainEditor",
  components: { ModalWindow, StatusBar, MainCanvas, SideBar },
  props: {
    img: String,
    state: String,
  },
  data() {
    return {
      originalWidth: 0,
      originalHeight: 0,

      canvasRef: null,
      newImg: null,
      origImg: null,
      isShowPanel: false,
      isShowCorrection: false,
      isShowFiltration: false,

      resizeUnit: "pixels",
      iw: null,
      ih: null,
      newiw: null,
      newih: null,
      bind: false,
      interpolation: "",
      isShowModal: false,

      xMouse: null,
      yMouse: null,
      pickedColor: "",
      scale: 100,
    };
  },
  emits: ["changeState"],
  mounted() {
    this.canvasRef = this.$refs.canvas.getCanvasRef();
  },
  methods: {
    setOriginalImageSize(width, height) {
      this.originalWidth = width;
      this.originalHeight = height;
    },
    closeModal() {
      this.isShowModal = false;
      this.$emit("changeState", "");
    },
    closePanel() {
      this.isShowPanel = false;
      this.$emit("changeState", "");
    },
    closeCorrection() {
      this.isShowCorrection = false;
      this.$emit("changeState", "");
    },
    closeFiltration() {
      this.isShowFiltration = false;
      this.$emit("changeState", "");
    },
    updateImageSizes(iw, ih) {
      this.iw = iw;
      this.ih = ih;
    },
    updateColor(color) {
      this.pickedColor = color;
    },
    updateCoordinates([x, y]) {
      this.xMouse = x;
      this.yMouse = y;
    },
    handleConfirm() {
      this.scale = 100;
      const image = new Image();
      image.src = this.img;

      if (this.resizeUnit === "pixels") {
        image.width = this.newiw;
        image.height = this.newih;
      }

      if (this.resizeUnit === "percentage") {
        const { width, height } = this.origImg;
        let widthMultiplier = this.newiw / 100;
        let heightMultiplier = this.newih / 100;

        // Проверка на синхронизацию
        if (this.bind) {
          heightMultiplier = widthMultiplier;
          this.newih = this.newiw; // Обновляем значение высоты в процентах
        }

        image.width = width * widthMultiplier;
        image.height = height * heightMultiplier;

        console.log('Original image width:', width, 'Original image height:', height);
        console.log('Width multiplier:', widthMultiplier, 'Height multiplier:', heightMultiplier);
        console.log('New image width:', image.width, 'New image height:', image.height);
      }

      this.iw = image.width;
      this.ih = image.height;
      this.newImg = image;
      this.isShowModal = false;
      this.$emit("changeState", "");
    },
    updateModal() {
      if (this.resizeUnit === "pixels") {
        if (this.newiw === null) {
          this.newiw = this.iw;
          if (this.canvasRef.width < this.iw) {
            this.newiw = this.canvasRef.width;
          }
        }

        if (this.newih === null) {
          this.newih = this.ih;
          if (this.canvasRef.height < this.ih) {
            this.newih = this.canvasRef.height;
          }
        }
      }

      if (this.resizeUnit === "percentage") {
        const { width, height } = this.origImg;
        const widthPercent = width / 100;
        const heightPercent = height / 100;

        if (this.previousWidthPixels === null && this.previousHeightPixels === null) {
          this.previousWidthPixels = this.iw;
          this.previousHeightPixels = this.ih;
        }

        if (this.newiw !== 100 && this.newih !== 100) {
          this.newiw = 100;
          this.newih = 100;
        }

        if (this.newiw == null) {
          this.newiw = ~~(this.iw / widthPercent);
          this.newih = ~~(this.ih / heightPercent);
        }
      }
    },
    updateNewiw(event) {
      const num = +event.target.value;

      if (!isNaN(num) && num > 0) {
        this.newiw = num;

        if (this.canvasRef.width < num) {
          this.newiw = this.canvasRef.width;
        }

        if (this.bind) {
          this.newih = this.newiw; // Привязка высоты к ширине
        }
      } else {
        this.newiw += 1;
        this.newiw -= 1;
      }
    },
    updateNewih(event) {
      const num = +event.target.value;

      if (!isNaN(num) && num > 0) {
        this.newih = num;

        if (this.canvasRef.height < num) {
          this.newih = this.canvasRef.height;
        }

        if (this.bind) {
          this.newiw = this.newih; // Привязка ширины к высоте
        }
      } else {
        this.newih += 1;
        this.newih -= 1;
      }
    },
    getResolutionNow() {
      return ((this.iw * this.ih) / 1000000).toFixed(2);
    },
    getResolutionAfter() {
      if (this.resizeUnit === "pixels") {
        return ((this.newiw * this.newih) / 1000000).toFixed(2);
      }
      if (this.resizeUnit === "percentage") {
        const { width, height } = this.origImg;
        const widthMultiplier = this.newiw / 100;
        const heightMultiplier = this.newih / 100;

        return (((width * widthMultiplier) * (height * heightMultiplier)) / 1000000).toFixed(2);
      }
    },
    interpolationCb(img, iw, ih) {
      if (this.interpolation === "nearestNeighbor") {
        return nearestNeighborInterpolation(img, iw, ih);
      }
      return null;
    },
    closeModals() {
      this.isShowPanel = false;
      this.isShowModal = false;
      this.isShowCorrection = false;
      this.isShowFiltration = false;
    },
    revertNewImg() {
      const clear = new Image();
      clear.width = this.newImg.width;
      clear.height = this.newImg.height;
      clear.src = this.newImg.src;
      this.newImg = clear;
    },
    updateNewImgData(data) {
      this.newImg.data = data;
    },
  },
  watch: {
    state(newValue) {
      if (newValue === "modal") {
        this.closeModals();
        this.isShowModal = true;
      }
      if (newValue === "pipette") {
        this.closeModals();
        this.isShowPanel = true;
      }
      if (newValue === "save") {
        this.handleConfirm();
      }
      if (newValue === "correct") {
        this.closeModals();
        this.isShowCorrection = true;
      }
      if (newValue === "filter") {
        this.closeModals();
        this.isShowFiltration = true;
      }
      if (!newValue) {
        this.closeModals();
      }
    },
    img(newVal) {
      if (newVal) {
        this.iw = null;
        this.ih = null;
        this.newiw = null;
        this.newih = null;
        this.scale = 100;

        this.origImg = new Image();
        this.origImg.src = this.img;
        this.newImg = this.origImg;
      }
    },
    isShowModal(newValue) {
      if (newValue) {
        this.updateModal();
      }
    },
  },
});
export function nearestNeighborInterpolation(img, newWidth, newHeight) {
  const originalWidth = img.width;
  const originalHeight = img.height;
  const scaleX = originalWidth / newWidth;
  const scaleY = originalHeight / newHeight;
  const newData = new Uint8ClampedArray(newWidth * newHeight * 4);

  for (let y = 0; y < newHeight; y++) {
    for (let x = 0; x < newWidth; x++) {
      const px = Math.floor(x * scaleX);
      const py = Math.floor(y * scaleY);
      const index = (y * newWidth + x) * 4;
      const originalIndex = (py * originalWidth + px) * 4;

      newData[index] = img.data[originalIndex];
      newData[index + 1] = img.data[originalIndex + 1];
      newData[index + 2] = img.data[originalIndex + 2];
      newData[index + 3] = img.data[originalIndex + 3];
    }
  }
  return new ImageData(newData, newWidth, newHeight);
}
</script>

<style scoped>
p {
  margin: 0;
}

.modal {
  padding: 15px;
  background: #f9f9f9;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.confirm-button {
  cursor: pointer;
  height: 35px;
  background: #007bff;
  color: #ffffff;
  border: none;
  border-radius: 5px;
  display: block;
}

.confirm-button:hover {
  background: #0056b3;
}

.resize-unit {
  height: 35px;
  width: 120px;
  border-radius: 5px;
  border: 1px solid #ccc;
  padding: 0 10px;
}

.sizes {
  display: flex;
  margin-top: 10px;
  width: 100%;
  align-items: flex-end;
  gap: 20px;
}

.custom-checkbox {
  position: relative;
  display: inline-block;
}

.custom-checkbox input,
.custom-checkbox .checkbox-image {
  width: 24px;
  height: 24px;
  cursor: pointer;
}

.custom-checkbox input[type="checkbox"] {
  position: absolute;
  opacity: 0;
  cursor: pointer;
}

.input-with-prefix {
  position: relative;
  display: flex;
  align-items: center;
  flex-direction: column;
}

.input-with-prefix .input-field {
  height: 35px;
  width: 80px;
  border: 1px solid #ccc;
  border-radius: 5px;
  padding: 0 10px;
  margin-left: 5px;
  margin-right: 5px;
}

.input-with-prefix .input-label {
  color: #6c757d;
  font-size: 14px;
  margin-left: 5px;
}

.input-with-prefix .input-suffix {
  color: #6c757d;
  font-size: 14px;
  margin-right: 5px;
}

.interpolation {
  display: flex;
  justify-content: space-between;
  margin-top: 10px;
}

.interpolation-select {
  height: 35px;
  border-radius: 5px;
  border: 1px solid #ccc;
  padding: 0 10px;
}

.sizes-counter {
  margin-top: 10px;
  display: flex;
  gap: 10px;
  justify-content: space-between;
}
.editor{
  display: flex;
  justify-content: space-between;
}
</style>