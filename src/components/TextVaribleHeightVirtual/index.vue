<template>
  <div
    ref="viewportRef"
    :style="{ height: AllHeight + 'px' }"
    class="viewport"
    @scroll="handleScroll"
  >
    <div
      class="content-placeholder"
      :style="{ height: `${renderTotalHeight}px` }"
    ></div>
    <div class="content-wrap" :style="{ transform: `translateY(${offset}px)` }">
      <div
        v-for="item in renderItems"
        :ref="(el) => renderItemsRef(el, item[props.keyField])"
        :key="item[props.keyField]"
        class="content-item"
      >
        <slot :item="item" />
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick, computed } from "vue";
const props = defineProps({
  allItems: {
    type: Array,
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
  renderSize: {
    type: Number,
    default: 15, // 每次渲染的 item 数量
  },
  buffer: {
    type: Number,
    default: 5,
  },
});
const cacheHeight = new Map(); // 缓存每个 item 的高度
const minItemHeight = ref(null);
const renderTotalHeight = computed(() => {
  console.log("asjfboea", minItemHeight.value);
  if (minItemHeight.value === null)
    return 0; // 如果没有测量过任何 item 的高度，返回 0
  else return minItemHeight.value * props.allItems.length;
});
const viewportRef = ref(null);
const renderItems = ref([]); // 当前需要渲染的 item
const offset = ref(0);
// 更新已渲染的 item 高度
const updateRenderItems = () => {
  const scrollTop = viewportRef.value?.scrollTop;

  let startIndex = 0;
  let startOffset = 0;
  for (let i = 0; i < props.allItems.length; i++) {
    const h = cacheHeight.get(props.allItems[i][props.keyField]);
    startOffset += h;
    if (startOffset >= scrollTop) {
      startIndex = i;
      break;
    }
  }

  renderItems.value = props.allItems.slice(
    startIndex,
    startIndex + props.renderSize + props.buffer
  );
  offset.value =
    startOffset - cacheHeight.get(props.allItems[startIndex][props.keyField]);
};

const renderItemsRef = (el, id) => {
  if (el) {
    // 存放已渲染的 item 的高度
    if (!cacheHeight.has(id)) {
      // 计算并缓存该 item 的高度
      cacheHeight.set(id, el.offsetHeight);
      if (minItemHeight.value === null) minItemHeight.value = el.offsetHeight;
      // 更新最小高度
      else minItemHeight.value = Math.min(minItemHeight.value, el.offsetHeight);
    }
  }
};
const handleScroll = () => {
  updateRenderItems();
};
onMounted(() => {
  updateRenderItems();
});
</script>

<style>
.viewport {
  overflow-y: auto;
  position: relative;
}
.content-placeholder {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
}
.content-item {
  border: 1px solid #000;
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>
