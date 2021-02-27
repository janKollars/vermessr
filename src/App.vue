<template>
  <aside :class="[ 'settings', { 'settings--operatable': image.dataUrl }]">
    <label for="gray-input">Gray</label><br>
    <input id="gray-input" v-model.number="searchArea" type="number" />
    <fieldset class="input-array">
      <legend>Overlay Position</legend>
      <label for="overlay-width">Width (%)</label>
      <input id="overlay-width" v-model.number="overlay.width" type="number" step="0.1" />
      <label for="overlay-height">Height (%)</label>
      <input id="overlay-height" v-model.number="overlay.height" type="number" step="0.1" />
      <label for="overlay-x">X (%)</label>
      <input id="overlay-x" v-model.number="overlay.x" type="number" step="0.1" />
      <label for="overlay-y">Y (%)</label>
      <input id="overlay-y" v-model.number="overlay.y" type="number" step="0.1" />
      <button :aria-pressed="resizing.toString()" title="Shift" @click="resizing = !resizing">resize freely</button>
    </fieldset>
    {{ foundValue }}
  </aside>

  <div :class="['canvas', { 'canvas--operating': image.loading }]" @drop="fileDropHandler" @dragover="fileDragOverHandler">
    <span v-if="!image.loading" class="canvas__drop-info">Drop image</span>
    <template v-else>
      <div class="uploaded-image" :style="image.width ? `--aspect-ratio: ${image.width}/${image.height}` : ''">
        <img v-if="image.dataUrl" :src="image.dataUrl" alt="" draggable="false" />
        <grid-overlay v-model:found-value="foundValue" v-model:overlay-position="overlay" :resize="resizing" :v-line="vLine" />
      </div>
    </template>
  </div>
</template>

<script>
import { computed, onMounted, reactive, ref } from 'vue'
import GridOverlay from './components/GridOverlay.vue'

export default {
  components: {
    GridOverlay
  },
  setup() {
    const foundValue = ref(undefined)
    const resizing = ref(false)
    const searchArea = ref(0)
    const vLine = computed(() =>
      searchArea.value * 10 / 8
    )

    const image = reactive({
      loading: false,
      dataUrl: undefined,
      width: undefined,
      height: undefined
    })
    const fileDropHandler = (event) => {
      event.preventDefault()
      const firstItem = event.dataTransfer.items[0]
      if (!firstItem?.type.startsWith('image/')) return
      const file = firstItem.getAsFile()
      const reader = new FileReader()
      reader.onload = (event) => {
        image.dataUrl = event.target.result
        const img = new Image()
        img.src = event.target.result
        img.onload = (() => {
          image.width = img.width
          image.height = img.height
        })
      }
      image.loading = true
      reader.readAsDataURL(file)
    }
    const fileDragOverHandler = (event) => {
      event.preventDefault()
    }

    const overlay = reactive({
      width: 100,
      height: 100,
      x: 0,
      y: 0
    })

    onMounted(() => {
      document.addEventListener('keydown', (event) => {
        if (event.code === 'ShiftLeft') {
          resizing.value = true
        }
      })
      document.addEventListener('keyup', (event) => {
        if (event.code === 'ShiftLeft') {
          resizing.value = false
        }
      })
    })

    return {
      foundValue,
      resizing,
      searchArea,
      vLine,
      image,
      fileDropHandler,
      fileDragOverHandler,
      overlay
    }
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  height: 100vh;
  font-family: sans-serif;
  background-color: #444444;
  color: #eeeeee;
}

input[type="number"] {
  width: 10ch;
}

input[type="number"], button {
  background-color: #3f3f3f;
  box-shadow: inset 0 0 2px currentColor;
  border: 1px solid currentColor;
  border-radius: 5px;
  padding: 0.25rem;
  color: inherit;
  outline: none;
}

input[type="number"]:focus-visible, button:focus-visible {
  outline: auto;
}

button[aria-pressed="true"] {
  background-color: #131313;
  box-shadow: inset 0 0 6px currentColor;
}

#app {
  display: grid;
  width: 100%;
  height: 100%;
  grid-template-columns: max-content 1fr;
  grid-gap: 1rem;
  padding: 1rem;
}

.settings:not(.settings--operatable) {
  visibility: hidden;
}

.input-array {
  display: grid;
  grid-template-columns: auto auto;
  grid-gap: 0.5rem;
  padding: 0.5rem;
}
.input-array > legend {
  grid-column: 1 / span 2;
}
.input-array > button {
  grid-column: span 2;
}

.canvas {
  position: relative;
  overflow: hidden;
  user-select: none;
  background-color: #333333;
}

.canvas--operating {
  background-image:
    linear-gradient(45deg, #646464 25%, transparent 25%),
    linear-gradient(-45deg, #646464 25%, transparent 25%),
    linear-gradient(45deg, transparent 75%, #646464 75%),
    linear-gradient(-45deg, transparent 75%, #646464 75%);
  background-size: 20px 20px;
  background-position: 0 0, 0 10px, 10px -10px, -10px 0px;
}

.canvas:not(.canvas--operating) {
  outline: deepskyblue;
  outline-width: 5px;
  outline-offset: -5px;
  outline-style: dashed;
}

.canvas__drop-info {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: 3rem;
  color: deepskyblue;
}

.uploaded-image {
  position: relative;
  aspect-ratio: var(--aspect-ratio);
}
.uploaded-image > img {
  width: 100%;
}
</style>