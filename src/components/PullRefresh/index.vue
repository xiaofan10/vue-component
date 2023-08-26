<template>
  <div ref="root" class="pull-refresh">
    <div
      ref="track"
      class="pull-refresh-track"
      :style="refreshStyleComputed"
      @touchstart.passive="onTouchStart"
      @touchend="onTouchEnd"
      @touchcancel="onTouchEnd"
    >
      <div ref="refresh" class="pull-refresh-track_refresh">
        <div v-if="TIPS_STATUS.includes(state.status)" class="refresh-text">
          {{ getStatusText() }}
        </div>
      </div>
      <slot></slot>
    </div>
  </div>
</template>

<script setup lang="ts">
import {
  onMounted,
  ref,
  computed,
  reactive,
  nextTick,
  unref,
  onUnmounted,
  onDeactivated,
  watch
} from 'vue'
import { useTouch } from './index'

type PullRefreshStatus = 'normal' | 'loading' | 'loosing' | 'pulling' | 'success'

const props = defineProps({
  successText: String,
  pullingText: String,
  loosingText: String,
  loadingText: String,
  modelValue: Boolean,
  pullDistance: [Number, String],
  headHeight: {
    type: [Number, String],
    default: 50
  },

  animationDuration: {
    type: [Number, String],
    default: 300
  }
})

const emit = defineEmits(['change', 'refresh', 'update:modelValue'])

const root = ref<HTMLElement | null>(null)
const track = ref<HTMLElement | null>(null)
const refresh = ref<HTMLElement | null>(null)
const load = ref<HTMLElement | null>(null)
// 滚动节点
const scrollEl = ref<HTMLElement | null>(null)
let reachTop: Boolean // 是否到达顶部

function isElement(node: Element) {
  const ELEMENT_NODE_TYPE = 1
  return node.tagName !== 'HTML' && node.tagName !== 'BODY' && node.nodeType === ELEMENT_NODE_TYPE
}

const overflowScrollReg = /scroll|auto|overlay/i

const getScrollParent = (el: HTMLElement): Element => {
  let node = el
  // 从root节点往上找，直到找到overflowY为scroll或者auto的节点
  while (node && isElement(node)) {
    const { overflowY } = window.getComputedStyle(node)
    if (overflowScrollReg.test(overflowY)) {
      return node
    } else {
      node = node.parentNode as HTMLElement
    }
  }
  // 这里body样式要设置成overflow:auto，其实应该返回window
  return document.body
}

const TEXT_STATUS = ['pulling', 'loosing', 'success']
const TIPS_STATUS = ['pulling', 'loosing', 'success', 'loading']
const TIPS_TEXT = {
  pulling: props.pullingText || '下拉刷新',
  loosing: props.loosingText || '松开刷新',
  success: props.successText || '刷新成功',
  loading: props.loadingText || '正在刷新'
}

const state = reactive({
  status: 'normal' as PullRefreshStatus,
  distance: 0,
  duration: 0
})

const setStatus = (distance: number, isLoading?: boolean) => {
  const pullDistance = +(props.pullDistance || props.headHeight)
  state.distance = distance

  if (isLoading) {
    state.status = 'loading'
  } else if (distance === 0) {
    state.status = 'normal'
  } else if (distance < pullDistance) {
    state.status = 'pulling'
  } else {
    state.status = 'loosing'
  }

  emit('change', {
    status: state.status,
    distance
  })
}

const refreshStyleComputed = computed(() => {
  return {
    transitionDuration: `${state.duration}ms`,
    transform: state.distance ? `translate3d(0,${state.distance}px, 0)` : ''
  }
})

const getStatusText = () => {
  const { status } = state
  if (status === 'normal') {
    return ''
  }
  return props[`${status}Text` as const] || TIPS_TEXT[status]
}

// 允许下拉的条件
const isTouchable = () => state.status !== 'loading' && state.status !== 'success'
const touch = useTouch()
const checkPosition = (event: TouchEvent) => {
  const el = scrollEl.value as HTMLElement
  const top = el.scrollTop
  reachTop = Math.max(top, 0) === 0
  // 如果页面元素在顶端，触发下拉刷新动作
  if (reachTop) {
    state.duration = 0
    touch.start(event)
  }
}

const ease = (distance: number) => {
  // 当移动距离大于 pullDistance 且小于2*pullDistance 时,有缓动，位移会多出（多出距离的一半）
  // 如果超出2*pullDistance,位移会为另一个缓动计算的距离
  const pullDistance = +(props.pullDistance || props.headHeight)
  if (distance > pullDistance) {
    if (distance < pullDistance * 2) {
      distance = pullDistance + (distance - pullDistance) / 2
    } else {
      distance = pullDistance * 1.5 + (distance - pullDistance * 2) / 4
    }
  }
  // 四舍五入取整
  return Math.round(distance)
}

const onTouchStart = (e: TouchEvent) => {
  console.log(reachTop,'realTop')
  // 非loading与success状态检测位置
  if (isTouchable()) {
    // scrollTop是否为0检查，如果为0，防止touchMove上拉触发下拉刷新bug
    checkPosition(e)
  }
}
const onTouchMove = (e: TouchEvent) => {
  // 非loading与success状态检测位置
  if (isTouchable()) {
    // 如果scrollTop不为0触发位置检测
    if (!reachTop) {
      checkPosition(e)
    }

    const { deltaY } = touch
    touch.move(e)
    // move到顶部（即scrollTop为0，并且touch移动位置大于0，且是垂直移动）时，改变下拉刷新位置
    if (reachTop && deltaY.value >= 0 && touch.isVertical()) {
      e?.preventDefault()
      setStatus(ease(deltaY.value))
    }
  }
}
const onTouchEnd = (e: TouchEvent) => {
  // 当松开手时，已经在顶部，手指移动值大于0且不是loading与success状态
  if (reachTop && touch.deltaY.value && isTouchable()) {
    state.duration = +props.animationDuration

    if (state.status === 'loosing') {
      setStatus(+props.headHeight, true)
      // 变更v-model值的变量为true
      emit('update:modelValue', true)
      // 触发自定义refresh事件
      nextTick(() => emit('refresh'))
    } else {
      // 否则下拉刷新位置归0
      setStatus(0)
    }
  }
}

onMounted(() => {
  scrollEl.value = getScrollParent(root.value as HTMLElement) as HTMLElement
})

watch(
  () => props.modelValue,
  (value) => {
    state.duration = +props.animationDuration

    if (value) {
      setStatus(+props.headHeight, true)
    } else if (props.successText) {
      state.status = 'success'

      setTimeout(() => {
        setStatus(0)
      }, 500)
    } else {
      setStatus(0, false)
    }
  }
)

let cleaned = false
let attached: boolean

const add = (target, type, listener) => {
  if (cleaned) {
    return
  }
  const element = unref(target)

  if (element && !attached) {
    element.addEventListener(type, listener, {
      capture: false,
      passive: false
    })
    attached = true
  }
}

const remove = (target, type, listener) => {
  if (cleaned) {
    return
  }
  const element = unref(target)

  if (element && attached) {
    element.removeEventListener(type, listener, false)
    attached = false
  }
}

onUnmounted(() => remove(track, 'touchmove', onTouchMove))
onDeactivated(() => remove(track, 'touchmove', onTouchMove))
onMounted(() => {
  add(track, 'touchmove', onTouchMove)
})
</script>

<style lang="less" scoped>
.pull-refresh {
  overflow: hidden;
  &-track {
    position: relative;
    height: 100%;
    transition-property: transform;
    &_refresh {
      display: flex;
      align-items: center;
      justify-content: center;
      position: absolute;
      left: 0;
      width: 100%;
      height: 50px;
      overflow: hidden;
      color: #ccc;
      font-size: 16px;
      transform: translateY(-100%);
    }
  }
}
</style>
