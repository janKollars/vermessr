<template>
  <div
    ref="el"
    :class="['grid-overlay', { 'grid-overlay--resizing': resize }]"
    :style="overlayVariables"
  >
    <svg
      :viewBox="`${scale.xStart} ${scale.yStart} ${scale.xEnd} ${scale.yEnd}`"
      preserveAspectRatio="none"
      @mouseover="!resize ? findValue($event) : ''"
      @click="!resize ? captureValue() : ''"
    >
      <line
        v-if="vRuler.position"
        :x1="vRuler.position"
        :y1="scale.yStart"
        :x2="vRuler.position"
        :y2="scale.yEnd"
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
        :stroke="hoveredValue === linesCount - hLine ? '#00000055' : 'transparent'"
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

<script>
import { computed, ref, unref, watch } from "vue";

export default {
  props: {
    foundValue: {
      type: Number,
      default: undefined,
    },
    overlayPosition: {
      type: Object,
      required: true,
    },
    resize: {
      type: Boolean,
      required: true,
    },
    scale: {
      type: Object,
      required: true,
    },
    vRuler: {
      type: Object,
      required: true,
    },
  },
  emits: ['update:found-value'],
  setup(props, { emit }) {
    const el = ref(null);
    watch(el, (el) => {
      const resizeObserver = new ResizeObserver((entries) => {
        startPosition.parentWidth = entries[0].contentRect.width;
        startPosition.parentHeight = entries[0].contentRect.height;
      });
      resizeObserver.observe(el.parentElement);
    });

    const overlayVariables = computed(() => ({
      '--overlay-width': props.overlayPosition.width + '%',
      '--overlay-height': props.overlayPosition.height + '%',
      '--overlay-x': props.overlayPosition.x + '%',
      '--overlay-y': props.overlayPosition.y + '%',
    }));
    const linesCount = computed(
      () => props.scale.yEnd - props.scale.yStart + 1
    );

    const hoveredValue = ref(undefined);
    const captureValue = () => {
      emit('update:found-value', hoveredValue.value);
    };
    const findValue = (event) => {
      hoveredValue.value =
        linesCount.value - 1 - event.target.getAttribute('y1');
    };

    const dragging = ref(false);

    const startPosition = {
      overlayPosition: { ...unref(props.overlayPosition) },
      mouseX: ref(undefined),
      mouseY: ref(undefined),
      parentWidth: undefined,
      parentHeight: undefined,
    };

    const mouseDownHandler = (event) => {
      startPosition.overlayPosition = { ...unref(props.overlayPosition) };
      startPosition.mouseX = event.x;
      startPosition.mouseY = event.y;
      dragging.value = true;
    };
    const mouseUpHandler = () => {
      dragging.value = false;
    };

    const registerMouseMoveEvent = (fun) => {
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
      x(event) {
        props.overlayPosition.x =
          startPosition.overlayPosition.x +
          ((event.x - startPosition.mouseX) / startPosition.parentWidth) * 100;
      },
      y(event) {
        props.overlayPosition.y =
          startPosition.overlayPosition.y +
          ((event.y - startPosition.mouseY) / startPosition.parentHeight) * 100;
      },
      width(event, additive = true) {
        props.overlayPosition.width =
          startPosition.overlayPosition.width +
          ((event.x - startPosition.mouseX) / startPosition.parentWidth) *
            100 *
            (additive ? 1 : -1);
      },
      height(event, additive = true) {
        props.overlayPosition.height =
          startPosition.overlayPosition.height +
          ((event.y - startPosition.mouseY) / startPosition.parentHeight) *
            100 *
            (additive ? 1 : -1);
      },
    };

    const transformHandler = {
      move(event) {
        event.preventDefault();
        transformHelper.y(event);
        transformHelper.x(event);
      },
      tl(event) {
        event.preventDefault();
        transformHelper.y(event);
        transformHelper.height(event, false);
        transformHelper.x(event);
        transformHelper.width(event, false);
      },
      t(event) {
        event.preventDefault();
        transformHelper.y(event);
        transformHelper.height(event, false);
      },
      tr(event) {
        event.preventDefault();
        transformHelper.y(event);
        transformHelper.height(event, false);
        transformHelper.width(event);
      },
      r(event) {
        event.preventDefault();
        transformHelper.width(event);
      },
      br(event) {
        event.preventDefault();
        transformHelper.height(event);
        transformHelper.width(event);
      },
      b(event) {
        event.preventDefault();
        transformHelper.height(event);
      },
      bl(event) {
        event.preventDefault();
        transformHelper.height(event);
        transformHelper.x(event);
        transformHelper.width(event, false);
      },
      l(event) {
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

    return {
      el,
      overlayVariables,
      linesCount,
      hoveredValue,
      findValue,
      captureValue,
      dragging,
      mouseDownHandler,
      mouseUpHandler,
      moveStart,
      resizeTLStart,
      resizeTStart,
      resizeTRStart,
      resizeRStart,
      resizeBRStart,
      resizeBStart,
      resizeBLStart,
      resizeLStart,
    };
  },
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
