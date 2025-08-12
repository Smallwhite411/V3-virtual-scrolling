<template>
  <div
    ref="container"
    class="virtual-list"
    @scroll.passive="onScroll"
    :style="{ overflowY: 'auto', height: containerHeight + 'px' }"
  >
    <!-- 上方占位 -->
    <div :style="{ height: paddingTop + 'px' }"></div>

    <!-- 可见内容 -->
    <div
      v-for="(item, index) in visibleData"
      :key="getKey(item, startIndex + index)"
      class="virtual-list-item"
      :style="{ height: itemHeight + 'px' }"
    >
      <slot :item="item" :index="startIndex + index"></slot>
    </div>

    <!-- 下方占位 -->
    <div :style="{ height: paddingBottom + 'px' }"></div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, type PropType } from "vue";

interface DefaultItem {
  id?: string | number;
  [key: string]: any;
}

const props = defineProps<{
  items: DefaultItem[];
  itemHeight: number;
  containerHeight: number; //当前容易的高度
  buffer?: number; //缓冲区（以containerHeight为200，itemHeight为50为例，200/50=4+buffter传的是5=9，也就是总共渲染9个）默认为2
  keyField?: string;
}>();
// 获取dom，计算滚动位置
const container = ref<HTMLElement | null>(null);
const scrollTop = ref(0);
// 以itemHeight为基准，计算当前视图可见的item数量 200/50=4
const visibleCount = computed(
  () => Math.ceil(props.containerHeight / props.itemHeight) //四舍五入进1
);
// 计算起始索引（当前视图顶部的索引）
const startIndex = computed(
  () => Math.floor(scrollTop.value / props.itemHeight) //四舍五入去1
);
// 计算结束索引（当前视图底部的索引）
const endIndex = computed(
  () =>
    Math.min(
      props.items.length,
      startIndex.value + visibleCount.value + (props.buffer || 2)
    ) //4+5 = 9
);
// 计算可见数据
const visibleData = computed(() =>
  props.items.slice(startIndex.value, endIndex.value)
);
// 上方占位符高度
const paddingTop = computed(() => startIndex.value * props.itemHeight);
// 下方占位符高度
const paddingBottom = computed(
  () => (props.items.length - endIndex.value) * props.itemHeight
);
// key字段设置为动态，可以自定义
const getKey = (item: DefaultItem, index: number) => {
  return props.keyField ? item[props.keyField] : index;
};

const onScroll = () => {
  if (container.value) {
    // 获取滚动位置（顶部距当前位置）
    scrollTop.value = container.value.scrollTop;
  }
};

onMounted(() => {
  if (container.value) {
    scrollTop.value = container.value.scrollTop;
  }
});
</script>

<style scoped>
.virtual-list {
  position: relative;
  width: 100%;
}

.virtual-list-item {
  box-sizing: border-box;
  border-bottom: 1px solid #eee;
  display: flex;
  align-items: center;
  padding: 0 8px;
}
</style>
