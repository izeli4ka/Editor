<template>
  <div class="container">
    <MainMenu
      @onImageSelected="handleImageSelected"
      :state="state"
      @changeState="changeState"
    />
    <MainEditor :img="img" @changeState="changeState" :state="state" />
  </div>
</template>

<script>
import MainMenu from './components/MainMenu.vue'
import MainEditor from './components/MainEditor.vue'

export default {
  name: 'App',
  components: {
    MainMenu,
    MainEditor,
  },
  data() {
    return {
      img: "",
      state: "",
    };
  },
  methods: {
    handleImageSelected(imageUrl) {
      this.img = imageUrl;
    },
    changeState(state) {
      this.state = state;
    },
    handleEscapeKey(event) {
      if (event.key === "Escape") {
        this.state = "";
      }
    },
  },
  mounted() {
    window.addEventListener("keydown", this.handleEscapeKey);
  },
  beforeUnmount() {
    window.removeEventListener("keydown", this.handleEscapeKey);
  },
}
</script>

<style scoped>
.container {
  /* display: flex; */
  width: 100%;
  height: 100%;
}
</style>
