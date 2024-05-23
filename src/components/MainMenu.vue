<template>
  <div class="menu">
    <button
      class="menu-button"
      @click="handleButtonClick"
      value="input"
      title="Choose an image file using the file selection dialog"
    >
      <label for="fileInput" class="button-label">
        <img class="button-image" src="../assets/file.png" alt="Select Image" />
      </label>
      <input
        id="fileInput"
        class="button-input"
        type="file"
        @change="handleImageSelection"
      />
    </button>
    <button
      class="menu-button"
      @click="handleButtonClick"
      :disabled="!selectedImage"
      value="pipette"
      title="Use the pipette tool to select a color from the image"
    >
      <label
        class="button-label"
        :class="{
          'button-label-tapped': state === 'pipette',
          disabled: !selectedImage,
        }"
      >
        <img class="button-image" src="../assets/pipette.png" alt="Pipette" />
      </label>
    </button>
    <button
      class="menu-button"
      @click="handleButtonClick"
      :disabled="!selectedImage"
      value="modal"
      title="Display a modal window on the image for inputting additional information or performing a specific action."
    >
      <label
        class="button-label"
        :class="{
          'button-label-tapped': state === 'modal',
          disabled: !selectedImage,
        }"
      >
        <img class="button-image" src="../assets/modal.png" alt="modal" />
      </label>
    </button>
    <button
      class="menu-button"
      @click="handleButtonClick"
      :disabled="!selectedImage"
      value="hand"
      title="Move the image across the canvas"
    >
      <label
        class="button-label"
        :class="{
          'button-label-tapped': state === 'hand',
          disabled: !selectedImage,
        }"
      >
        <img class="button-image" src="../assets/hand.png" alt="Move Image" />
      </label>
    </button>
    <button
      class="menu-button"
      @click="handleButtonClick"
      :disabled="!selectedImage"
      value="save"
      title="Save the current image after making any necessary edits"
    >
      <label class="button-label" :class="{ disabled: !selectedImage }">
        <img
          class="button-image"
          src="../assets/downloads.png"
          alt="Save Image"
        />
      </label>
    </button>
    <button
      class="menu-button"
      @click="handleButtonClick"
      :disabled="!selectedImage"
      value="correct"
      title="Edit the current image"
    >
      <label class="button-label" :class="{ disabled: !selectedImage }">
        <img
          class="button-image"
          src="../assets/correction.svg"
          alt="Correct Image"
        />
      </label>
    </button>
    <button
      class="menu-button"
      @click="handleButtonClick"
      :disabled="!selectedImage"
      value="filter"
      title="Filter the current image"
    >
      <label class="button-label" :class="{ disabled: !selectedImage }">
        <img
          class="button-image"
          src="../assets/filtration.svg"
          alt="Filter Image"
        />
      </label>
    </button>
    <div class="url">
      <input
        class="url-input"
        v-model="imageUrl"
        placeholder="URL"
        @keydown.enter="loadImageFromUrl"
      />
    </div>
  </div>
</template>

<script>
import { defineComponent } from "vue";

export default defineComponent({
  name: "MainMenu",
  props: {
    state: String,
  },
  data() {
    return {
      selectedImage: "",
      imageUrl: "",
    };
  },
  methods: {
    handleImageSelection(event) {
      const target = event.target;
      const selectedFile = target.files?.[0];

      if (selectedFile) {
        const imageUrl = URL.createObjectURL(selectedFile);
        this.selectedImage = imageUrl;
        this.$emit("onImageSelected", imageUrl);

        target.value = "";
      }
    },
    loadImageFromUrl() {
      if (this.imageUrl) {
        this.$emit("onImageSelected", this.imageUrl);
        this.selectedImage = this.imageUrl;
        this.imageUrl = "";
      }
    },
    handleButtonClick(event) {
      const target = event.target;
      const value = target.parentElement?.value
        ? target.parentElement.value
        : target.parentElement?.parentElement.value;
      this.$emit("changeState", value);
    },
  },
  beforeUnmount() {
    if (this.selectedImage) {
      URL.revokeObjectURL(this.selectedImage);
    }
  },
  watch: {
    state(newValue) {
      if (newValue === "pipette") {
        this.stateCursor = "pipette";
      } else this.stateCursor = null;
    },
  },
});
</script>

<style scoped>
.menu-button {
  position: relative;
  display: inline-block;
  cursor: pointer;
  border: none;
  padding: 5px;
  margin: 0;
  background-color: #ffffff;
}
.disabled {
  opacity: 0.1;

}

.button-label {
  display: flex;
}

.button-image {
  width: 35px; /* Установите размер изображения по вашему выбору */
  height: auto; /* Автоматическое масштабирование высоты */
}

.button-input {
  display: none; /* Скрываем элемент input */
}

/* Эффект при наведении */
.menu-button:hover {
  background-color: #00d9ff; /* Затемненный фон */
  border-radius: 30%;
}

.menu {
  /* width: 50px; */
  height: calc(100% - 10px);
  padding: 5px;
  display: flex;
  flex-direction: row;
  gap: 5px;
}

.url {
  display: flex;
  align-items: center;
  margin-left: 20px;
}
.url-input {
  flex: 1; /* Растягиваем ввод по всей доступной ширине */
  padding: 8px; /* Отступы вокруг текстового поля */
  font-size: 16px; /* Размер шрифта */
  border: 1px solid #ccc; /* Граница текстового поля */
  border-radius: 4px; /* Скругленные углы */
  outline: none; /* Убираем обводку при фокусе */
}
.url-input::placeholder {
  color: #999; /* Цвет плейсхолдера */
}
.url-input:focus {
  border-color: #00d9ff; /* Цвет границы при фокусе */
  box-shadow: 0 0 5px #00d9ff; /* Тень при фокусе */
}
</style>
