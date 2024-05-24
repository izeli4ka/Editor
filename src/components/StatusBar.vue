<template>
  <div :class="{ 'status-bar': true, 'pipette-mode': state === 'pipette' && pickedColor }">
    <div class="left-side">
      <span v-if="state"> Status: {{ state }}</span>
      <span v-if="imageWidth && imageHeight"> | W: {{ formattedImageWidth }} | H: {{ formattedImageHeight }}</span>
      <span v-if="state === 'pipette' && pickedColor"> | C: {{ pickedColor }}</span>
      <div v-if="state === 'pipette' && pickedColor" class="pipette-color" :style="{ background: pickedColor }"></div>
      <span v-if="state === 'pipette' && pickedColor && xMouse && yMouse"> | Coords: {{ xMouse }}:{{ yMouse }}</span>
    </div>

    <div class="right-side">
      <input
        type="range"
        class="scale"
        id="scale"
        name="scale"
        min="12"
        max="300"
        step="1"
        :value="scale"
        @input="updateScale"
      />
      <span>Scale: {{ scale }}</span>
    </div>
  </div>
</template>

<script>
import { defineComponent } from "vue";

export default defineComponent({
  name: "StatusBar",
  props: {
    state: String,
    imageHeight: Number,
    imageWidth: Number,
    pickedColor: String,
    xMouse: Number,
    yMouse: Number,
    scale: Number,
  },
  computed: {
    formattedImageWidth() {
      return Math.floor(this.imageWidth);
    },
    formattedImageHeight() {
      return Math.floor(this.imageHeight);
    }
  },
  methods: {
    updateScale(event) {
      const newScale = +event.target.value;
      this.$emit("updateScale", newScale);
    },
  },
});
</script>

<style scoped>
.status-bar {
  position: absolute;
  bottom: 0;
  /* width: 100%; */
  /* height: 30px; */
  /* background: #e5e5e5; */
  padding: 10px;
}

.left-side {
  display: flex;
}

.pipette-color {
  margin-top: 2.5px;
  width: 15px;
  height: 15px;
}

.scale {
  display: block;
}
</style>
