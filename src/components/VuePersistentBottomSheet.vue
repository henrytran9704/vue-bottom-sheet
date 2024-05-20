<template>
  <Teleport to="body">
    <div
      class="bottom-sheet"
      v-bind="$attrs"
      ref="bottomSheet"
      :aria-hidden="!showSheet"
      role="dialog"
    >
      <transition>
        <div
          @click="clickOnOverlayHandler"
          class="bottom-sheet__overlay"
          v-show="overlay && showSheet"
        />
      </transition>
      <div ref="bottomSheetContent" :class="sheetContentClasses">
        <header ref="bottomSheetHeader" class="bottom-sheet__header">
          <div class="bottom-sheet__draggable-area" ref="bottomSheetDraggableArea">
            <div class="bottom-sheet__draggable-thumb"></div>
          </div>

          <slot name="header" />
        </header>
        <main ref="bottomSheetMain" class="bottom-sheet__main">
          <slot />
        </main>
        <footer ref="bottomSheetFooter" class="bottom-sheet__footer">
          <slot name="footer" />
        </footer>
      </div>
    </div>
  </Teleport>
</template>

<script setup lang="ts">
import { computed, onMounted, nextTick, ref, watch } from 'vue'
import Hammer from 'hammerjs'

type Position = 'hide' | 'middle' | 'full'
/**
 * Bottom sheet props interface
 */
interface IProps {
  overlay?: boolean
  overlayColor?: string
  maxWidth?: number
  maxHeight?: number
  transitionDuration?: number
  overlayClickClose?: boolean
  canSwipe?: boolean
  height?: Position
  classStyle?: string
}

/**
 * Touch event interface
 */
interface IEvent {
  type: string
  deltaY: number
  isFinal: boolean
  cancelable?: boolean
}

/**
 * Bottom sheet props interface
 */
const props = withDefaults(defineProps<IProps>(), {
  overlay: true,
  overlayColor: '#0000004D',
  maxWidth: 640,
  transitionDuration: 0.5,
  overlayClickClose: true,
  canSwipe: true,
  height: 'hide'
})

/**
 * Bottom sheet emit interface
 */
const emit = defineEmits(['opened', 'closed', 'dragging-up', 'dragging-down'])

/**
 * Show or hide sheet
 */
const showSheet = ref(true)

/**
 * Sheet height value
 */
const sheetHeight = ref(0)

const sheetHeaderHeight = ref(0)

/**
 * Dynamic translate value
 */
const translateValue = ref(80)

/**
 * Flag to check if sheet is being dragged
 */
const isDragging = ref(false)

/**
 * Content scrolled value
 */
const contentScroll = ref(0)

const transitionDurationLocal = ref(0)

const middlePoint = ref(false)

/**
 * Refs to all sheet HTML elements
 */
const bottomSheet = ref<HTMLElement | null>(null)
const bottomSheetHeader = ref<HTMLElement | null>(null)
const bottomSheetMain = ref<HTMLElement | null>(null)
const bottomSheetFooter = ref<HTMLElement | null>(null)
const bottomSheetContent = ref<HTMLElement | null>(null)
const bottomSheetDraggableArea = ref<HTMLElement | null>(null)
const positionElm = ref<Position>(props.height)
const positionElmHeight = ref<70 | 0 | 30>(70)
const direction = ref(0)

watch(
  () => props.height,
  () => {
    showSheet.value = false
    console.log('watching props.height')
    isDragging.value = true
    positionElm.value = props.height
    positionToVh(positionElm.value)
    isDragging.value = false
    console.log(translateValue.value)

    showSheet.value = true
  },
  { immediate: true }
)
/**
 * Close bottom sheet when escape key is pressed
 * @param element
 */
const isFocused = (element: HTMLElement) => document.activeElement === element
window.addEventListener('keyup', (event: KeyboardEvent) => {
  const isSheetElementFocused =
    bottomSheet.value!.contains(event.target as HTMLElement) &&
    isFocused(event.target as HTMLElement)

  if (event.key === 'Escape' && !isSheetElementFocused) {
    close()
  }
})

/**
 * Return all classes for bottom sheet content
 */
const sheetContentClasses = computed(() => {
  return [
    'bottom-sheet__content',
    {
      'bottom-sheet__content--fullscreen': sheetHeight.value >= window.innerHeight,
      'bottom-sheet__content--dragging': isDragging.value
    }
  ]
})

/**
 * Return transition duration value with seconds
 */
const transitionDurationString = computed(() => {
  return `${transitionDurationLocal.value}s`
})

/**
 * Return sheet height string with px
 */
const sheetHeightString = computed(() => {
  return sheetHeight.value && sheetHeight.value > 0 ? `${sheetHeight.value + 1}px` : 'auto'
})

/**
 * Return max height string
 */
const maxHeightString = computed(() => {
  return props.maxHeight ? `${props.maxHeight}px` : 'inherit'
})

/**
 * Return current translate value string with percents
 */
const translateValueString = computed(() => {
  if (translateValue.value === 85) {
    return `${sheetHeight.value - sheetHeaderHeight.value}px`
  }
  return `${(sheetHeight.value * translateValue.value) / 100}px`
})

/**
 * Return max width string
 */
const maxWidthString = computed(() => {
  return `${props.maxWidth}px`
})

/**
 * Calculate sheet height
 */
const initHeight = async () => {
  await nextTick()
  sheetHeaderHeight.value = bottomSheetHeader.value!.offsetHeight
  sheetHeight.value =
    bottomSheetHeader.value!.offsetHeight +
    bottomSheetMain.value!.clientHeight +
    bottomSheetFooter.value!.offsetHeight
}

/**
 * Move sheet while dragging
 * @param event
 * @param type
 */

const dragHandler = (event: IEvent, type: 'area' | 'main') => {
  if (props.canSwipe) {
    isDragging.value = true

    const preventDefault = (e: Event) => {
      e.preventDefault()
    }

    if (type === 'area') {
      translateValue.value = pixelToVh(event.deltaY)
    }

    if (event.type === 'panup') {
      emit('dragging-up')
    } else if (event.type === 'pandown') {
      emit('dragging-down')
    }

    const a = event.type
    if (a == 'panup' || a == 'pandown') {
      direction.value = a == 'panup' ? 1 : -1
    }

    // Drag down end
    if (event.isFinal && direction.value === -1) {
      const xx = translateValue.value

      isDragging.value = false

      if (middlePoint.value) {
        if (xx < 10) {
          translateValue.value = 0
          positionElm.value = 'full'
          bottomSheetMain.value?.setAttribute(
            'style',
            `height:${(props.maxHeight || 0) * (1 - 0) - sheetHeaderHeight.value}px`
          )
        } else if (xx >= 10 && xx <= 45) {
          translateValue.value = 45
          positionElm.value = 'middle'
          bottomSheetMain.value?.setAttribute(
            'style',
            `height:${(props.maxHeight || 0) * (1 - 0.45) - sheetHeaderHeight.value}px`
          )
        } else if (xx > 45) {
          translateValue.value = 85
          positionElm.value = 'hide'
          bottomSheetMain.value?.setAttribute(
            'style',
            `height:${(props.maxHeight || 0) * (1 - 0.85) - sheetHeaderHeight.value}px`
          )
        }
      } else {
        if (xx < 10) {
          translateValue.value = 0
          positionElm.value = 'full'
        } else if (xx >= 10) {
          translateValue.value = 85
          positionElm.value = 'hide'
        }
      }

      event.deltaY = 0
    }

    // Drag up end
    if (event.isFinal && direction.value === 1) {
      if (type === 'main') {
        contentScroll.value = bottomSheetMain.value!.scrollTop
      }

      isDragging.value = false

      const xx = translateValue.value

      if (middlePoint.value) {
        if (xx < 45) {
          translateValue.value = 0
          positionElm.value = 'full'
          bottomSheetMain.value?.setAttribute(
            'style',
            `height:${(props.maxHeight || 0) * (1 - 0) - sheetHeaderHeight.value}px`
          )
        } else if (xx < 85) {
          translateValue.value = 45
          positionElm.value = 'middle'
          bottomSheetMain.value?.setAttribute(
            'style',
            `height:${(props.maxHeight || 0) * (1 - 0.45) - sheetHeaderHeight.value}px`
          )
        }
      } else {
        if (xx < 85) {
          translateValue.value = 0
          positionElm.value = 'full'
        }
      }

      event.deltaY = 0
    }
  }
}

nextTick(() => {
  /**
   * Set initial card height
   */
  initHeight()

  /**
   * Create instances of Hammerjs
   */
  const hammerAreaInstance = new Hammer(bottomSheetHeader.value as HTMLElement, {
    inputClass: Hammer.TouchMouseInput,
    recognizers: [[Hammer.Pan, { direction: Hammer.DIRECTION_VERTICAL }]]
  })

  // const hammerMainInstance = new Hammer(bottomSheetMain.value as HTMLElement, {
  //   inputClass: Hammer.TouchMouseInput,
  //   recognizers: [[Hammer.Pan, { direction: Hammer.DIRECTION_VERTICAL }]]
  // })

  /**
   * Set events and handlers to hammerjs instances
   */
  hammerAreaInstance.on('panstart panup pandown panend', (e: IEvent) => {
    dragHandler(e, 'area')
  })

  // hammerMainInstance.on('panstart panup pandown panend', (e: IEvent) => {
  //   dragHandler(e, 'main')
  // })
})

/**
 * Open bottom sheet method
 */
const open = () => {
  // positionToVh(positionElm.value)

  translateValue.value = 85

  // document.documentElement.style.overflowY = 'hidden'
  // document.documentElement.style.overscrollBehavior = 'none'
  showSheet.value = true
  emit('opened')
}

function positionToVh(heigth: Position) {
  if (heigth == 'full') {
    translateValue.value = 0
  } else if (heigth == 'middle') {
    translateValue.value = 45
  } else {
    translateValue.value = 85
  }
}

/**
 * Close bottom sheet method
 */
const close = async () => {
  showSheet.value = false
  translateValue.value = 85
  setTimeout(() => {
    // document.documentElement.style.overflowY = 'auto'
    // document.documentElement.style.overscrollBehavior = ''
    emit('closed')
  }, transitionDurationLocal.value * 1000)
}

/**
 * Click on overlay handler
 */
const clickOnOverlayHandler = () => {
  console.log('clickOnOverlayHandler')

  // if (props.overlayClickClose) {
  //   close()
  // }
}

/**
 * Convert pixels to vh
 * @param pixel
 */
const pixelToVh = (pixel: number) => {
  const height =
    props.maxHeight && props.maxHeight <= sheetHeight.value ? props.maxHeight : sheetHeight.value

  // console.log('{height}')
  // console.log({ height })

  let x = (pixel / height) * 100

  if (positionElm.value == 'hide') {
    x = x + 85
  }
  if (positionElm.value == 'middle') {
    x = x + 45
  }
  if (positionElm.value == 'full') {
    x = x + 0
  }
  if (x < 0) {
    x = 0
  }
  return x
}

/**
 * Define public methods
 */
defineExpose({ open, close })

onMounted(() => {
  // open()
  setTimeout(() => {
    transitionDurationLocal.value = props.transitionDuration
  }, 0)
})
</script>

<style lang="scss" scoped>
.bottom-sheet {
  pointer-events: none;
  z-index: 100;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-end;

  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  transition: visibility v-bind('transitionDurationString');

  * {
    box-sizing: border-box;
  }

  &[aria-hidden='false'] {
    visibility: visible;
  }

  &[aria-hidden='true'] {
    visibility: hidden;
    pointer-events: none;
  }

  &__overlay {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: -1;
    background: v-bind('props.overlayColor');
  }

  &__content {
    box-shadow: 0px -2px 8px 0px #0000001a;
    display: flex;
    flex-direction: column;
    border-radius: 16px 16px 0 0;
    background: #ffffff;
    overflow-y: hidden;
    transform: translate3d(0, v-bind('translateValueString'), 0);
    height: v-bind('sheetHeightString');
    max-width: v-bind('maxWidthString');
    width: 100%;
    max-height: v-bind('maxHeightString');
    box-sizing: border-box;
    pointer-events: all;

    &--fullscreen {
      // border-radius: 0;
    }

    &:not(.bottom-sheet__content--dragging) {
      transition: v-bind('transitionDurationString') ease;
    }
  }

  &__header {
    cursor: grab;
  }

  &__draggable-area {
    width: 100%;
    margin: auto;
    padding: 8px;
  }

  &__draggable-thumb {
    width: 80px;
    height: 3px;
    background: #d9d9d9;
    border-radius: 8px;
    margin: 0 auto;
  }

  &__main {
    display: flex;
    flex-direction: column;
    overflow-y: scroll;
    box-sizing: border-box;
    -webkit-overflow-scrolling: touch;
    touch-action: auto !important;

    &::-webkit-scrollbar {
      height: 8px;
      width: 8px;
    }
    &::-webkit-scrollbar-corner {
      display: none;
    }
    &:hover::-webkit-scrollbar-thumb {
      background-color: rgba(0, 0, 0, 0.2);
      border-radius: 8px;
    }
    &::-webkit-scrollbar-thumb {
      background-color: rgba(0, 0, 0, 0);
    }
  }

  &__footer:empty {
    display: none;
  }
}

.v-enter-active,
.v-leave-active {
  transition: opacity v-bind('transitionDurationString') ease;
}

.v-enter-from,
.v-leave-to {
  opacity: 0;
}
</style>
