<template>
  <div
    ref="containerRef"
    class="virtual-list"
    @scroll.passive="onScrollHandler"
    :style="{ overflowY: 'auto', height: `${height}px`, position: 'relative' }"
  >
    <!-- 占位高度 -->
    <div :style="{ height: `${totalHeight}px` }"></div>

    <!-- 可见区域内容 -->
    <div
      :style="{
        position: 'absolute',
        top: `${startOffset}px`,
        left: 0,
        right: 0,
      }"
    >
      <div
        v-for="(item, index) in visibleData"
        :key="startIndex + index"
        :style="{
          height: `${itemHeight}px`,
          boxSizing: 'border-box',
          display: 'flex',
          justifyContent: 'center',
          alignItems: 'center',
          borderBottom: '1px solid #eee',
        }"
      >
        <slot :item="item" :index="startIndex + index"></slot>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted, nextTick, defineExpose } from "vue";

interface Props<T> {
  items: T[];
  itemHeight: number;
  height: number; //固定高度
  buffer?: number;
}

const props = withDefaults(defineProps<Props<any>>(), {
  buffer: 5, // 默认缓冲区
});

const containerRef = ref<HTMLElement | null>(null);
const scrollTop = ref(0);
const startIndex = ref(0);
const startOffset = ref(0);

let ticking = false; // 用于 rAF 节流

// 总高度
const totalHeight = computed(() => props.items.length * props.itemHeight);

// 一屏可渲染的数量
const visibleCount = computed(() => Math.ceil(props.height / props.itemHeight));

// 计算可见数据
const visibleData = computed(() => {
  const endIndex = startIndex.value + visibleCount.value + props.buffer;
  return props.items.slice(startIndex.value, endIndex);
});

// 滚动事件（rAF 节流）
const onScrollHandler = () => {
  if (!ticking) {
    window.requestAnimationFrame(() => {
      const scrollTopVal = containerRef.value?.scrollTop || 0;
      scrollTop.value = scrollTopVal;
      startIndex.value = Math.floor(scrollTopVal / props.itemHeight);
      startOffset.value = startIndex.value * props.itemHeight;
      ticking = false;
    });
    ticking = true;
  }
};

// API: 滚动到指定项
const scrollToIndex = (index: number) => {
  if (!containerRef.value) return;
  const top = index * props.itemHeight;
  containerRef.value.scrollTop = top;
};

// 对外暴露方法
defineExpose({
  scrollToIndex,
});

onMounted(() => {
  nextTick(() => {
    onScrollHandler(); // 初始化计算一次
  });
});
</script>

<style scoped>
.virtual-list {
  width: 100%;
}
</style>
