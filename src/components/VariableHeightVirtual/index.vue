<template>
  <div
    ref="scrollEl"
    class="text-viral"
    :style="{
      height: AllHeight + 'px',
      overflowY: 'auto',
      position: 'relative',
    }"
    @scroll.passive="onScroll"
  >
    <!-- 用总高度撑起滚动条 -->
    <div :style="{ height: totalHeight + 'px', position: 'relative' }">
      <!-- 仅渲染可见区；每项按累计偏移绝对定位 -->
      <div
        v-for="(item, vIdx) in visibleItems"
        :key="getKey(item, startIndex + vIdx)"
        :style="{
          position: 'absolute',
          top: positions[startIndex + vIdx] + 'px',
          left: 0,
          right: 0,
        }"
        :ref="(el) => bindItemEl(el as HTMLElement | null, startIndex + vIdx)"
        class="text-list-item"
      >
        <slot :item="item" :index="startIndex + vIdx"></slot>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import {
  computed,
  ref,
  watch,
  nextTick,
  onMounted,
  onBeforeUnmount,
  withDefaults,
  defineProps,
  defineExpose,
} from "vue";

interface DefaultItem {
  id?: string | number;
  [k: string]: any;
}

type Props = {
  data: DefaultItem[];
  keyField?: string;
  AllHeight: number; // 视口高度
  estimatedItemHeight?: number; // 未测量前的估算高度
  buffer?: number; // 额外缓冲条目数
  // 可选：无限滚动守卫
  hasMore?: boolean;
  loading?: boolean;
  threshold?: number; // 距底多少像素触发 loadMore
};

const props = withDefaults(defineProps<Props>(), {
  keyField: "id",
  AllHeight: 300,
  estimatedItemHeight: 48,
  buffer: 6,
  hasMore: false,
  loading: false,
  threshold: 200,
});

const emit = defineEmits<{ (e: "loadMore"): void }>();

const scrollEl = ref<HTMLDivElement | null>(null);

// —— 高度 & 累计偏移 —— //
const heights = ref<number[]>([]); // index -> 实高（或估算）
const positions = ref<number[]>([]); // index -> 顶部累计偏移
const key2height = new Map<any, number>(); // key -> 最近一次实测高度缓存
const totalHeight = ref(0);

const getKey = (item: DefaultItem, index: number) =>
  props.keyField && item?.[props.keyField] != null
    ? item[props.keyField]
    : index;

// 确保容量并用估值/历史实测填充
const ensureCapacity = () => {
  const len = props.data.length;
  const est = props.estimatedItemHeight!;
  const next = new Array<number>(len);
  for (let i = 0; i < len; i++) {
    const k = getKey(props.data[i], i);
    const known = key2height.get(k);
    next[i] = known ?? heights.value[i] ?? est;
  }
  heights.value = next;
  recomputePositions();
};

// O(n) 重算累计偏移 + 总高度
const recomputePositions = () => {
  const arr = heights.value;
  positions.value.length = arr.length;
  if (!arr.length) {
    totalHeight.value = 0;
    return;
  }
  positions.value[0] = 0;
  for (let i = 1; i < arr.length; i++) {
    positions.value[i] = positions.value[i - 1] + arr[i - 1];
  }
  totalHeight.value = positions.value[arr.length - 1] + arr[arr.length - 1];
};

// —— 二分定位 startIndex —— //
const findStartIndex = (scrollTop: number) => {
  let lo = 0,
    hi = positions.value.length - 1,
    ans = 0;
  while (lo <= hi) {
    const mid = (lo + hi) >> 1;
    if (positions.value[mid] <= scrollTop) {
      ans = mid;
      lo = mid + 1;
    } else hi = mid - 1;
  }
  return ans;
};

const startIndex = ref(0);
// 根据真实高度向后覆盖到一屏，再追加 buffer
const endIndex = computed(() => {
  const len = props.data.length;
  if (!len) return 0;
  let i = startIndex.value;
  let covered = 0;
  const need = props.AllHeight;
  while (i < len && covered < need) {
    covered += heights.value[i] ?? props.estimatedItemHeight!;
    i++;
  }
  return Math.min(len, i + props.buffer!);
});

const visibleItems = computed(() =>
  props.data.slice(startIndex.value, endIndex.value)
);

// —— 单一的 rAF 节流滚动处理器 —— //
let ticking = false;
let suppressScroll = false; // 程序性滚动抑制标记
let lastLoadTotal = 0; // 防重复 loadMore（同一总高度只触发一次）

const maybeEmitLoadMore = (top: number) => {
  if (!props.hasMore || props.loading) return;
  const nearBottom =
    top + props.AllHeight >= totalHeight.value - props.threshold!;
  if (nearBottom && totalHeight.value !== lastLoadTotal) {
    lastLoadTotal = totalHeight.value;
    emit("loadMore");
  }
};

const onScroll = () => {
  if (suppressScroll) return;
  if (ticking) return;
  ticking = true;
  requestAnimationFrame(() => {
    const top = scrollEl.value?.scrollTop ?? 0;
    startIndex.value = findStartIndex(top);
    maybeEmitLoadMore(top);
    ticking = false;
  });
};

// —— 观察可见项高度（含宽度变化带来的换行） —— //
const roMap = new Map<number, ResizeObserver>();
const bindItemEl = (el: HTMLElement | null, index: number) => {
  // 清理旧 observer（避免重复绑定）
  roMap.get(index)?.disconnect();
  roMap.delete(index);
  if (!el) return;

  const ro = new ResizeObserver((entries) => {
    for (const entry of entries) {
      const h = Math.max(1, Math.round(entry.contentRect.height)); // 防 0
      if (h !== heights.value[index]) {
        // 锚点补偿：保持当前 startIndex 顶部在视口的位置不跳
        const oldTop = positions.value[startIndex.value] ?? 0;

        heights.value[index] = h;
        const k = getKey(props.data[index], index);
        key2height.set(k, h);
        recomputePositions();

        const newTop = positions.value[startIndex.value] ?? 0;
        const delta = newTop - oldTop;
        if (delta !== 0 && scrollEl.value) {
          suppressScroll = true;
          scrollEl.value.scrollTop += delta;
          // 释放抑制（微任务，确保这次滚动事件被忽略）
          queueMicrotask(() => {
            suppressScroll = false;
          });
        }
      }
    }
  });
  ro.observe(el);
  roMap.set(index, ro);
};

// —— 容器 ResizeObserver：浏览器宽度变化时，依赖 item 的 RO 自动重测 —— //
let containerRO: ResizeObserver | null = null;
onMounted(() => {
  ensureCapacity();
  // 初始化一次 startIndex
  const top = scrollEl.value?.scrollTop ?? 0;
  startIndex.value = findStartIndex(top);

  containerRO = new ResizeObserver(() => {
    // 可见项高度变化会由各自的 RO 触发；这里补一次稳态重新定位
    requestAnimationFrame(() => {
      const t = scrollEl.value?.scrollTop ?? 0;
      startIndex.value = findStartIndex(t);
    });
  });
  if (scrollEl.value) containerRO.observe(scrollEl.value);
});

onBeforeUnmount(() => {
  containerRO?.disconnect();
  roMap.forEach((ro) => ro.disconnect());
  roMap.clear();
});

// 数据变化：复用 key 的历史实测高度，减少闪跳
watch(
  () => props.data,
  async () => {
    ensureCapacity();
    await nextTick();
    // 不强制触发滚动；由布局/RO 驱动
  },
  { deep: false }
);

// 暴露 API：滚动到指定索引
const scrollToIndex = (index: number) => {
  if (!scrollEl.value) return;
  const top = positions.value[index] ?? 0;
  suppressScroll = true;
  scrollEl.value.scrollTop = top;
  queueMicrotask(() => {
    suppressScroll = false;
  });
};
defineExpose({ scrollToIndex });
</script>

<style scoped lang="scss">
.text-viral {
  width: 100%;
  border: 1px solid #ccc;
  border-radius: 10px;
}
.text-list-item {
  box-sizing: content-box;
  border-bottom: 1px solid #eee;
  padding: 6px 8px;
  display: block; /* 宽度变化触发 RO，高度可被观测到 */
}
</style>
