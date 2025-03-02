<template>
  <div
    ref="dragRef"
    :class="['es-drager', { disabled, dragging: isMousedown, selected }]"
    :style="dragStyle"
    @click.stop
  >
    <slot />
    
    <template v-if="!disabled && resizable">
      <div
        v-for="item, index in dotList"
        :key="index"
        class="es-drager-dot"
        :data-side="item.side"
        :style="{ ...item }"
        @mousedown.stop.prevent="onDotMousedown(item, $event)"
      >
        <slot name="resize" v-bind="{ side: item.side }">
          <div class="es-drager-dot-handle"></div>
        </slot>
      </div>
    </template>
    
    <Rotate
      v-if="!disabled && selected"
      v-model="dragData.angle"
      :drag-data="dragData"
      :element="dragRef"
      @rotate="emitFn('rotate', dragData)"
      @rotate-start="emitFn('rotate-start', dragData)"
      @rotate-end="handleRotateEnd"
    >
      <slot name="rotate" />
    </Rotate>
  </div>
</template>

<script setup lang='ts'>
import { computed, ref, watch } from 'vue'
import {
  DragerProps,
  formatData,
  withUnit,
  getDotList,
  getLength,
  degToRadian,
  getNewStyle,
  centerToTL,
  EventType,
  calcGrid
} from './drager'
import { useDrager, setupMove } from './use-drager'
import Rotate from './rotate.vue'

const props = defineProps(DragerProps)
const emit = defineEmits(['change', 'drag', 'drag-start', 'drag-end', 'resize', 'resize-start', 'resize-end', 'rotate', 'rotate-start', 'rotate-end'])
const emitFn = (type: EventType, ...args: any) => {
  emit(type, ...args)
  emit('change', ...args)
}
const dragRef = ref<HTMLElement | null>(null)
const { selected, dragData, isMousedown } = useDrager(dragRef, props, emitFn)

const dotList = ref(getDotList())
// 果然，computed产生一个dragStyle，调整元素位置
const dragStyle = computed(() => {
  const { width, height, left, top, angle } = dragData.value
  return {
    width: withUnit(width),
    height: withUnit(height),
    left: withUnit(left),
    top: withUnit(top),
    transform: `rotate(${angle}deg)`,
    '--es-drager-color': props.color
  }
})

function handleRotateEnd(angle: number) {
  dotList.value = getDotList(angle)
  emitFn('rotate-end', dragData.value)
}

/**
 * 缩放
 * @param dotInfo 
 * @param e 
 */
function onDotMousedown(dotInfo: any, e: MouseEvent) {
  e.stopPropagation()
  e.preventDefault()
  // 获取鼠标按下的坐标
  const downX = e.clientX
  const downY = e.clientY
  const { width, height, left, top } = dragData.value

  // 中心点
  const centerX = left + width / 2
  const centerY = top + height / 2

  const rect = { width, height, centerX, centerY, rotateAngle: dragData.value.angle }
  const type = dotInfo.side

  const { minWidth, minHeight, aspectRatio } = props
  emitFn('resize-start', dragData.value)

  const onMousemove = (e: MouseEvent) => {

    const { clientX, clientY } = e
    // 距离
    let deltaX = (clientX - downX) / props.scaleRatio
    let deltaY = (clientY - downY) / props.scaleRatio
    // 开启网格缩放
    if (props.snapToGrid) {
      deltaX = calcGrid(deltaX, props.gridX)
      deltaY = calcGrid(deltaY, props.gridY)
    }

    const alpha = Math.atan2(deltaY, deltaX)
    const deltaL = getLength(deltaX, deltaY)
    const isShiftKey = e.shiftKey

    const beta = alpha - degToRadian(rect.rotateAngle)
    const deltaW = deltaL * Math.cos(beta)
    const deltaH = deltaL * Math.sin(beta)
    const ratio = isShiftKey && !aspectRatio ? rect.width / rect.height : aspectRatio
    
    const {
      position: { centerX, centerY },
      size: { width, height }
    } = getNewStyle(type, { ...rect, rotateAngle: rect.rotateAngle }, deltaW, deltaH, ratio, minWidth, minHeight)
   
    const pData = centerToTL({
      centerX,
      centerY,
      width,
      height,
      angle: dragData.value.angle
    })

    dragData.value = {
      ...dragData.value,
      ...formatData(pData, centerX, centerY)
    }
    emitFn('resize', dragData.value)
  }

  setupMove(onMousemove, () => {
    emitFn('resize-end', dragData.value)
  })
}

watch(() => [
  props.width,
  props.height,
  props.left,
  props.top,
  props.angle
], ([width, height, left, top, angle]) => {
  dragData.value = {
    ...dragData.value,
    width,
    height,
    left,
    top,
    angle
  }
})

watch(() => props.selected, (val) => {
  selected.value = val
}, { immediate: true })

</script>

<style lang='scss'>
.es-drager {
  position: absolute;
  z-index: 1000;
  width: 200px;
  height: 120px;
  border: 1px solid var(--es-drager-color, #3a7afe);
  
  &.selected {
    .es-drager-dot {
      display: block;
    }
  }
  > * {
    -webkit-user-drag: none;
    user-select: none;
  }
  &.dragging {
    user-select: none;
  }
  &.disabled {
    opacity: 0.4;
    cursor: not-allowed !important;
  }
  &:hover {
    cursor: move;
  }
  &-dot {
    display: none;
    position: absolute;
    transform: translate(-50%, -50%);
    cursor: se-resize;
    &[data-side*="right"] {
      transform: translate(50%, -50%);
    }
    &[data-side*="bottom"] {
      transform: translate(-50%, 50%);
    }
    &[data-side="bottom-right"] {
      transform: translate(50%, 50%);
    }

    &-handle {
      width: 10px;
      height: 10px;
      border-radius: 50%;
      background-color: var(--es-drager-color, #3a7afe);
    }
  }
}
</style>
