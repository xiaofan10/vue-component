<template>
  <div class="container">
    <PullRefresh v-model="isRefreshing" @refresh="onRefresh" success-text="刷新成功">
      <LoadMore v-model="isLoading" @load="onLoad" success-text="加载成功">
        <div class="list" v-for="(item, i) in list" :key="i" :style="{ backgroundColor: item }">
          {{ i }}列表内容
        </div>
      </LoadMore>
    </PullRefresh>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import PullRefresh from '@/components/PullRefresh/index.vue'
import LoadMore from '@/components/LoadMore/index.vue'

const list = ref(
  Array(15)
    .fill(1)
    .map((item) => {
      const r = Math.floor(Math.random() * 100000) % 255
      const g = Math.floor(Math.random() * 100000) % 255
      const b = Math.floor(Math.random() * 100000) % 255
      const a = Number(Math.random().toFixed(1))
      const color = `rgba(${r},${g},${b},${a})`
      return color
    })
)
const isLoading = ref(false)
const isRefreshing = ref(false)

const onRefresh = () => {
  console.log('触发了refresh')
  setTimeout(() => {
    isRefreshing.value = false
  }, 2000)
}

const onLoad = () => {
  console.log('触发了load')
  setTimeout(() => {
    isLoading.value = false
  }, 2000)
}
</script>

<style lang="less" scoped>
.container {
  width: 100vw;
  height: 100vh;
  overflow: auto;
  .list {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100px;
    margin-bottom: 5px;
    box-sizing: border-box;
  }
}
</style>
