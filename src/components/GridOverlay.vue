<template>
  <div
    ref="el"
    :class="['grid-overlay', { 'grid-overlay--resizing': resize }]"
    :style="overlayVariables"
  >
    <svg
      :viewBox="`${scale.xStart} 0 ${scale.xEnd} ${scale.yEnd - scale.yStart}`"
      preserveAspectRatio="none"
      @mouseover="!resize ? findValue($event) : ''"
      @click="!resize ? captureValue() : ''"
    >
      <line
        v-if="vRuler.position"
        :x1="vRuler.position"
        y1="0"
        :x2="vRuler.position"
        :y2="scale.yEnd - scale.yStart"
        stroke="black"
        :stroke-width="vRuler.size"
      />
      <line
        v-for="hLine in linesCount"
        :key="hLine"
        :x1="scale.xStart"
        :y1="hLine - 1"
        :x2="scale.xEnd"
        :y2="hLine - 1"
        :stroke="hoveredValue === linesCount + scale.yStart - hLine ? '#00000055' : 'transparent'"
        stroke-width="1"
      />
    </svg>
    <div
      v-if="resize"
      class="grab-areas"
      @mousedown="mouseDownHandler"
      @mouseup="mouseUpHandler"
    >
      <div class="grab-areas__top-left" @mousedown="resizeTLStart()" />
      <div class="grab-areas__top-center" @mousedown="resizeTStart()" />
      <div class="grab-areas__top-right" @mousedown="resizeTRStart()" />
      <div class="grab-areas__center-left" @mousedown="resizeLStart()" />
      <div class="grab-areas__center-center" @mousedown="moveStart()" />
      <div class="grab-areas__center-right" @mousedown="resizeRStart()" />
      <div class="grab-areas__bottom-left" @mousedown="resizeBLStart()" />
      <div class="grab-areas__bottom-center" @mousedown="resizeBStart()" />
      <div class="grab-areas__bottom-right" @mousedown="resizeBRStart()" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, computed, unref } from 'vue';

const props = defineProps<{
  foundValue?: number,
  overlayPosition: OverlayPositionSetting,
  resize: boolean,
  scale: ScaleSetting,
  vRuler: VRulerSetting,
}>()

const emit = defineEmits<{
  (e: 'update:found-value', value: number): void
}>()

const el = ref<HTMLElement | null>(null);
  const parentDimensions = {
  width: 0,
  height: 0
}
watch(el, (newValue) => {
  if (!newValue?.parentElement) return;
  const resizeObserver = new ResizeObserver((entries) => {
    parentDimensions.width = entries[0].contentRect.width;
    parentDimensions.height = entries[0].contentRect.height;
  });
  resizeObserver.observe(newValue.parentElement);
}, { immediate: true })

const overlayVariables = computed(() => ({
  '--overlay-width': props.overlayPosition.width + '%',
  '--overlay-height': props.overlayPosition.height + '%',
  '--overlay-x': props.overlayPosition.x + '%',
  '--overlay-y': props.overlayPosition.y + '%',
}));
const linesCount = computed(
  () => props.scale.yEnd - props.scale.yStart + 1
);

const hoveredValue = ref<number | undefined>(undefined);
const findValue = (event: MouseEvent) => {
  if (!event.target) return;
  const hoveredY1Value = Number((event.target as SVGLineElement).getAttribute('y1') as string);
  hoveredValue.value =
  linesCount.value - 1 - (hoveredY1Value - props.scale.yStart);
};
const captureValue = () => {
  if (hoveredValue.value === undefined) return;
  emit('update:found-value', hoveredValue.value);
};

const dragging = ref(false);

const startPosition = {
  overlayPosition: { ...unref(props.overlayPosition) },
  mouseX: 0,
  mouseY: 0,
};

const mouseDownHandler = (event: MouseEvent) => {
  startPosition.overlayPosition = { ...unref(props.overlayPosition) };
  startPosition.mouseX = event.x;
  startPosition.mouseY = event.y;
  dragging.value = true;
};
const mouseUpHandler = () => {
  dragging.value = false;
};

const registerMouseMoveEvent = (fun: (event: MouseEvent) => void) => {
  document.addEventListener('mousemove', fun);
  document.addEventListener(
    'mouseup',
    () => {
      document.removeEventListener('mousemove', fun);
    },
    {
      once: true,
    }
  );
};

const transformHelper = {
  x(event: MouseEvent) {
    props.overlayPosition.x =
      startPosition.overlayPosition.x +
      ((event.x - startPosition.mouseX) / parentDimensions.width) * 100;
  },
  y(event: MouseEvent) {
    props.overlayPosition.y =
      startPosition.overlayPosition.y +
      ((event.y - startPosition.mouseY) / parentDimensions.height) * 100;
  },
  width(event: MouseEvent, additive = true) {
    props.overlayPosition.width =
      startPosition.overlayPosition.width +
      ((event.x - startPosition.mouseX) / parentDimensions.width) *
        100 *
        (additive ? 1 : -1);
  },
  height(event: MouseEvent, additive = true) {
    props.overlayPosition.height =
      startPosition.overlayPosition.height +
      ((event.y - startPosition.mouseY) / parentDimensions.height) *
        100 *
        (additive ? 1 : -1);
  },
};

const transformHandler = {
  move(event: MouseEvent) {
    event.preventDefault();
    transformHelper.y(event);
    transformHelper.x(event);
  },
  tl(event: MouseEvent) {
    event.preventDefault();
    transformHelper.y(event);
    transformHelper.height(event, false);
    transformHelper.x(event);
    transformHelper.width(event, false);
  },
  t(event: MouseEvent) {
    event.preventDefault();
    transformHelper.y(event);
    transformHelper.height(event, false);
  },
  tr(event: MouseEvent) {
    event.preventDefault();
    transformHelper.y(event);
    transformHelper.height(event, false);
    transformHelper.width(event);
  },
  r(event: MouseEvent) {
    event.preventDefault();
    transformHelper.width(event);
  },
  br(event: MouseEvent) {
    event.preventDefault();
    transformHelper.height(event);
    transformHelper.width(event);
  },
  b(event: MouseEvent) {
    event.preventDefault();
    transformHelper.height(event);
  },
  bl(event: MouseEvent) {
    event.preventDefault();
    transformHelper.height(event);
    transformHelper.x(event);
    transformHelper.width(event, false);
  },
  l(event: MouseEvent) {
    event.preventDefault();
    transformHelper.x(event);
    transformHelper.width(event, false);
  },
};

const moveStart = () => {
  registerMouseMoveEvent(transformHandler.move);
};
const resizeTLStart = () => {
  registerMouseMoveEvent(transformHandler.tl);
};
const resizeTStart = () => {
  registerMouseMoveEvent(transformHandler.t);
};
const resizeTRStart = () => {
  registerMouseMoveEvent(transformHandler.tr);
};
const resizeRStart = () => {
  registerMouseMoveEvent(transformHandler.r);
};
const resizeBRStart = () => {
  registerMouseMoveEvent(transformHandler.br);
};
const resizeBStart = () => {
  registerMouseMoveEvent(transformHandler.b);
};
const resizeBLStart = () => {
  registerMouseMoveEvent(transformHandler.bl);
};
const resizeLStart = () => {
  registerMouseMoveEvent(transformHandler.l);
};
</script>

<style>
.grid-overlay {
  position: absolute;
  width: var(--overlay-width);
  height: var(--overlay-height);
  top: var(--overlay-y);
  left: var(--overlay-x);
  outline: fuchsia;
  outline-style: solid;
  outline-offset: -1px;
  outline-width: 1px;
}

.grid-overlay > svg {
  height: 100%;
  width: 100%;
}
.grid-overlay--resizing > svg {
  display: none;
}

.grab-areas {
  position: absolute;
  top: 0;
  left: 0;
  display: grid;
  grid-template-columns: 1.5rem 1fr 1.5rem;
  grid-template-rows: 1.5rem 1fr 1.5rem;
  height: 100%;
  width: 100%;
}

.grab-areas__top-center,
.grab-areas__bottom-center {
  cursor: ns-resize;
}
.grab-areas__center-left,
.grab-areas__center-right {
  cursor: ew-resize;
}
.grab-areas__top-left,
.grab-areas__bottom-right {
  cursor: nwse-resize;
}
.grab-areas__top-right,
.grab-areas__bottom-left {
  cursor: nesw-resize;
}
.grab-areas__center-center {
  cursor: move;
}
</style>
