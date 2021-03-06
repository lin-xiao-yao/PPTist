<template>
  <div class="hamster-ppt-screen">
    <div 
      class="slide-list"
      @mousewheel="$event => mousewheelListener($event)"
      v-contextmenu="contextmenus"
    >
      <div 
        :class="[
          'slide-item', 
          `turning-mode-${slide.turningMode || 'slideY'}`,
          {
            'current': index === slideIndex,
            'before': index < slideIndex,
            'after': index > slideIndex,
            'hide': (index === slideIndex - 1 || index === slideIndex + 1) && slide.turningMode !== currentSlide.turningMode,
          }
        ]"
        v-for="(slide, index) in slides" 
        :key="slide.id"
      >
        <div 
          class="slide-content" 
          :style="{
            width: slideWidth + 'px',
            height: slideHeight + 'px',
          }"
        >
          <ScreenSlide 
            :slide="slide" 
            :scale="scale"
            :animationIndex="animationIndex"
          />
        </div>
      </div>
    </div>

    <Modal
      v-model:visible="slideThumbnailModelVisible" 
      :footer="null" 
      centered
      :width="1020"
      :bodyStyle="{ padding: '50px 20px 20px 20px' }"
    >
      <SlideThumbnails :turnSlideToIndex="turnSlideToIndex" />
    </Modal>

    <div class="tools">
      <IconLeftC class="tool-btn" @click="execPrev()" />
      <IconRightC class="tool-btn" @click="execNext()" />
      <IconSearch class="tool-btn" @click="slideThumbnailModelVisible = true" />
      <Popover trigger="click" v-model:visible="writingBoardToolVisible">
        <template #content>
          <WritingBoardTool @close="writingBoardToolVisible = false" />
        </template>
        <IconWrite class="tool-btn" />
      </Popover>
    </div>
  </div>
</template>

<script lang="ts">
import { computed, defineComponent, onMounted, onUnmounted, provide, ref } from 'vue'
import throttle from 'lodash/throttle'
import { MutationTypes, useStore } from '@/store'
import { Slide } from '@/types/slides'
import { VIEWPORT_ASPECT_RATIO, VIEWPORT_SIZE } from '@/configs/canvas'
import { KEYS } from '@/configs/hotkey'
import { ContextmenuItem } from '@/components/Contextmenu/types'
import { isFullscreen } from '@/utils/fullscreen'
import useScreening from '@/hooks/useScreening'

import ScreenSlide from './ScreenSlide.vue'
import SlideThumbnails from './SlideThumbnails.vue'
import WritingBoardTool from './WritingBoardTool.vue'

export default defineComponent({
  name: 'screen',
  components: {
    ScreenSlide,
    SlideThumbnails,
    WritingBoardTool,
  },
  setup() {
    const store = useStore()
    const slides = computed(() => store.state.slides)
    const slideIndex = computed(() => store.state.slideIndex)
    const currentSlide = computed<Slide>(() => store.getters.currentSlide)

    const slideWidth = ref(0)
    const slideHeight = ref(0)

    const scale = computed(() => slideWidth.value / VIEWPORT_SIZE)

    const slideThumbnailModelVisible = ref(false)

    const writingBoardToolVisible = ref(false)

    const setSlideContentSize = () => {
      const winWidth = document.body.clientWidth
      const winHeight = document.body.clientHeight
      let width, height

      if (winHeight / winWidth === VIEWPORT_ASPECT_RATIO) {
        width = winWidth
        height = winHeight
      }
      else if (winHeight / winWidth > VIEWPORT_ASPECT_RATIO) {
        width = winWidth
        height = winWidth * VIEWPORT_ASPECT_RATIO
      }
      else {
        width = winHeight / VIEWPORT_ASPECT_RATIO
        height = winHeight
      }
      slideWidth.value = width
      slideHeight.value = height
    }

    const { exitScreening } = useScreening()

    const windowResizeListener = () => {
      setSlideContentSize()
      if (!isFullscreen()) exitScreening()
    }

    const animationIndex = ref(0)
    const animations = computed(() => currentSlide.value.animations || [])

    const runAnimation = () => {
      const prefix = 'animate__'
      const animation = animations.value[animationIndex.value]
      animationIndex.value += 1

      const elRef = document.querySelector(`#screen-element-${animation.elId} [class^=base-element-]`)
      if (elRef) {
        const animationName = `${prefix}${animation.type}`
        elRef.classList.add(`${prefix}animated`, animationName)

        const handleAnimationEnd = () => {
          elRef.classList.remove(`${prefix}animated`, animationName)
        }
        elRef.addEventListener('animationend', handleAnimationEnd, { once: true })
      }
    }

    const execPrev = () => {
      if (animations.value.length && animationIndex.value > 0) {
        animationIndex.value -= 1
      }
      else if (slideIndex.value > 0) {
        store.commit(MutationTypes.UPDATE_SLIDE_INDEX, slideIndex.value - 1)
        const lastIndex = animations.value ? animations.value.length : 0
        animationIndex.value = lastIndex
      }
    }
    const execNext = () => {
      if (animations.value.length && animationIndex.value < animations.value.length) {
        runAnimation()
      }
      else if (slideIndex.value < slides.value.length - 1) {
        store.commit(MutationTypes.UPDATE_SLIDE_INDEX, slideIndex.value + 1)
        animationIndex.value = 0
      }
    }

    const keydownListener = (e: KeyboardEvent) => {
      const key = e.key.toUpperCase()
      if (key === KEYS.UP || key === KEYS.LEFT) execPrev()
      else if (
        key === KEYS.DOWN || 
        key === KEYS.RIGHT ||
        key === KEYS.SPACE || 
        key === KEYS.ENTER
      ) execNext()
    }

    const mousewheelListener = throttle(function (e: WheelEvent) {
      if (e.deltaY < 0) execPrev()
      else if (e.deltaY > 0) execNext()
    }, 500, { leading: true, trailing: false })

    onMounted(() => {
      window.addEventListener('resize', windowResizeListener)
      document.addEventListener('keydown', keydownListener)
    })
    onUnmounted(() => {
      window.removeEventListener('resize', windowResizeListener)
      document.removeEventListener('keydown', keydownListener)
    })

    const turnPrevSlide = () => {
      store.commit(MutationTypes.UPDATE_SLIDE_INDEX, slideIndex.value - 1)
      animationIndex.value = 0
    }
    const turnNextSlide = () => {
      store.commit(MutationTypes.UPDATE_SLIDE_INDEX, slideIndex.value + 1)
      animationIndex.value = 0
    }

    const turnSlideToIndex = (index: number) => {
      slideThumbnailModelVisible.value = false
      store.commit(MutationTypes.UPDATE_SLIDE_INDEX, index)
      animationIndex.value = 0
    }

    const contextmenus = (): ContextmenuItem[] => {
      return [
        {
          text: '上一页',
          disable: slideIndex.value <= 0,
          handler: () => turnPrevSlide(),
        },
        {
          text: '下一页',
          disable: slideIndex.value >= slides.value.length - 1,
          handler: () => turnNextSlide(),
        },
        {
          text: '结束放映',
          subText: 'ESC',
          handler: exitScreening,
        },
      ]
    }
    
    provide('slideScale', scale)

    return {
      slides,
      slideIndex,
      currentSlide,
      slideWidth,
      slideHeight,
      scale,
      mousewheelListener,
      animationIndex,
      contextmenus,
      execPrev,
      execNext,
      slideThumbnailModelVisible,
      turnSlideToIndex,
      writingBoardToolVisible,
    }
  },
})
</script>

<style lang="scss" scoped>
.hamster-ppt-screen {
  width: 100%;
  height: 100%;
  position: relative;
  background-color: #111;
}
.slide-list {
  background: #1d1d1d;
  position: relative;
  width: 100%;
  height: 100%;
}
.slide-item {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;

  &.current {
    z-index: 2;
  }

  &.hide {
    opacity: 0;
  }

  &.turning-mode-no {
    &.before {
      transform: translateY(-100%);
    }
    &.after {
      transform: translateY(100%);
    }
  }
  &.turning-mode-fade {
    transition: opacity .75s;
    &.before {
      pointer-events: none;
      opacity: 0;
    }
    &.after {
      pointer-events: none;
      opacity: 0;
    }
  }
  &.turning-mode-slideX {
    transition: transform .35s;
    &.before {
      transform: translateX(-100%);
    }
    &.after {
      transform: translateX(100%);
    }
  }
  &.turning-mode-slideY {
    transition: transform .35s;
    &.before {
      transform: translateY(-100%);
    }
    &.after {
      transform: translateY(100%);
    }
  }
}
.slide-content {
  background-color: #fff;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  display: flex;
  justify-content: center;
  align-items: center;
}

.tools {
  position: fixed;
  bottom: 8px;
  left: 8px;
  font-size: 25px;
  color: #666;
  z-index: 10;
  cursor: pointer;
}
.tool-btn {
  opacity: .35;

  &:hover {
    opacity: .7;
  }
  & + .tool-btn {
    margin-left: 8px;
  }
}
</style>