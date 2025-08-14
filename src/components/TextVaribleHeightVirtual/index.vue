<template>
  <div
    ref="viewportRef"
    :style="{ height: AllHeight + 'px' }"
    class="viewport"
    @scroll="handleScroll"
  >
    <div class="content-placeholder"></div>
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
import { ref, onMounted } from "vue";
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
const viewportRef = ref(null);
const renderItems = ref([]); // 当前需要渲染的 item
const hasRenderedItemsHeight = ref({}); // 已渲染的 item 数据 height
const offset = ref(0);
// 更新已渲染的 item 高度
const updateRenderItems = () => {
  const scrollTop = viewportRef.value?.scrollTop;

  let startIndex = 0;
  let startOffset = 0;

  for (let i = 0; i < props.allItems.length; i++) {
    const h = hasRenderedItemsHeight.value[props.allItems[i][props.keyField]];
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
    startOffset - hasRenderedItemsHeight.value[props.allItems[startIndex][props.keyField]];
};

const renderItemsRef = (el, id) => {
  if (el) {
    // 存放已渲染的 item 的高度
    hasRenderedItemsHeight.value[id] = el.offsetHeight;
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
