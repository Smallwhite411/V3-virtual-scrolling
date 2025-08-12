<template>
  <div
    ref="scrollContainer"
    class="text-viral"
    :style="{ height: AllHeight + 'px', overflowY: 'auto' }"
    @scroll.passive="onScroll"
  >
    <div :style="{ paddingTop: paddingTop + 'px' }"></div>
    <div
      v-for="(item, index) in currentData"
      :style="{ height: itemHeight + 'px' }"
      :key="getKey(item, startIndex + index)"
      class="text-list-item"
    >
      <slot :item="item" :index="startIndex + index"></slot>
    </div>
    <div :style="{ paddingBottom: paddingBottom + 'px' }"></div>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
interface Item {
  id: number;
  text: string;
}
interface DefaultItem {
  id?: string | number;
  [key: string]: any;
}
const props = defineProps({
  data: {
    type: Array<Item>,
    default: () => [],
  },
  keyField: {
    type: String,
    default: "id",
  },
  AllHeight: {
    type: Number,
    default: 200,
  },
  itemHeight: {
    type: Number,
    default: 40,
  },
  buffer: {
    type: Number,
    default: 5,
  },
});
const scrollTop = ref(0);
const scrollBottom = ref(0);
const startIndex = computed(() =>
  Math.floor(scrollTop.value / props.itemHeight)
);
const visibleCount = computed(() =>
  Math.ceil(props.AllHeight / props.itemHeight)
);
const endIndex = computed(() =>
  Math.min(
    props.data.length,
    startIndex.value + visibleCount.value + props.buffer
  )
);
const scrollContainer = ref<Element | null>(null);
const paddingTop = computed(() => startIndex.value * props.itemHeight);
const paddingBottom = computed(
  () => (props.data.length - endIndex.value) * props.itemHeight
);
const onScroll = () => {
  if (scrollContainer.value) scrollTop.value = scrollContainer.value.scrollTop;
};
// key字段设置为动态，可以自定义
const getKey = (item: DefaultItem, index: number) => {
  return props.keyField ? item[props.keyField] : index;
};
// 动态计算需要展示的item的数量
const currentData = computed(() =>
  props.data.slice(startIndex.value, endIndex.value)
);
onMounted(() => {
  if (scrollContainer.value) {
    scrollTop.value = scrollContainer.value.scrollTop;
  }
});
</script>

<style scoped lang="scss">
.text-viral {
  position: relative;
  width: 100%;
  border: 1px solid #ccc;
  border-radius: 10px;
}

.text-list-item {
  box-sizing: border-box;
  border-bottom: 1px solid #eee;
  display: flex;
  align-items: center;
  padding: 0 8px;
}
</style>
