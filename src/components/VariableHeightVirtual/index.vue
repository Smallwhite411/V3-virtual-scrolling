<template>
  <div
    ref="containerRef"
    class="virtual-list-container"
    :style="{ height: AllHeight + 'px', overflowY: 'auto' }"
    @scroll.passive="onScroll"
  >
    <!-- 整个内容高度，内部用绝对定位 -->
    <div :style="{ height: totalHeight + 'px', position: 'relative' }">
      <div
        v-for="idx in visibleIndices"
        :key="getKey(data[idx], idx)"
        class="virtual-item"
        :data-index="idx"
        :style="{
          position: 'absolute',
          top: offsets[idx] + 'px',
          left: 0,
          right: 0,
        }"
        :ref="(el) => setItemRef(el, idx)"
      >
        <slot :item="data[idx]" :index="idx"></slot>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import {
  ref,
  computed,
  onMounted,
  onBeforeUnmount,
  nextTick,
  watch,
} from "vue";
import type { PropType } from "vue";

type DefaultItem = Record<string, any>;

const props = defineProps({
  data: { type: Array as PropType<DefaultItem[]>, default: () => [] },
  keyField: { type: String, default: "id" },
  AllHeight: { type: Number, default: 300 },
  // 初始高度估值，尽量接近你列表里元素的平均高度
  initialItemHeight: { type: Number, default: 50 },
  buffer: { type: Number, default: 5 },
});

/* ---------------- state ---------------- */
const containerRef = ref<HTMLElement | null>(null);

const data = computed(() => props.data);
const n = computed(() => data.value.length);

// per-item heights (px). 未测量项保留估值
const itemHeights = ref<number[]>([]);
// offsets: prefix sum, offsets[0]=0; offsets[i] = sum heights[0..i-1]
// offsets.length === n+1 (方便 totalHeight = offsets[n])
const offsets = ref<number[]>([]);

const totalHeight = computed(() => offsets.value[n.value] ?? 0);

/* visible window */
const scrollTop = ref(0);
const startIndex = ref(0);
const endIndex = ref(0);

// list of indices to render
const visibleIndices = computed(() => {
  const res: number[] = [];
  for (let i = startIndex.value; i < endIndex.value; i++) res.push(i);
  return res;
});

/* DOM refs & observer */
const itemElements = new Map<number, HTMLElement>(); // index -> el
let ro: ResizeObserver | null = null;
const observed = new Map<number, HTMLElement>(); // tracked by RO

/* ---------------- helpers ---------------- */
function getKey(item: DefaultItem, index: number) {
  return props.keyField ? item?.[props.keyField] ?? index : index;
}

function ensureInitHeights() {
  // 如果数据长度改变，更新 itemHeights 长度并填充估值
  const len = n.value;
  if (itemHeights.value.length !== len) {
    const old = itemHeights.value.slice(0, len);
    const fill = new Array(Math.max(0, len - old.length)).fill(
      props.initialItemHeight
    );
    itemHeights.value = old.concat(fill);
  }
  // build offsets
  updateOffsets();
}

function updateOffsets() {
  const len = n.value;
  const offs: number[] = new Array(len + 1);
  let sum = 0;
  offs[0] = 0;
  for (let i = 0; i < len; i++) {
    sum += itemHeights.value[i] ?? props.initialItemHeight;
    offs[i + 1] = sum;
  }
  offsets.value = offs;
}

/** 二分查找：返回最大的 i 使 offsets[i] <= value */
function findStartIndexByScrollTop(value: number) {
  const offs = offsets.value;
  let low = 0;
  let high = Math.max(0, offs.length - 1);
  while (low <= high) {
    const mid = Math.floor((low + high) / 2);
    if ((offs[mid] ?? 0) <= value) {
      low = mid + 1;
    } else {
      high = mid - 1;
    }
  }
  return Math.max(0, low - 1);
}

/* 估算可视项数量（基于当前平均高度） */
const averageHeight = computed(() => {
  const arr = itemHeights.value;
  if (!arr.length) return props.initialItemHeight;
  const sum = arr.reduce((s, v) => s + (v ?? props.initialItemHeight), 0);
  return Math.max(1, sum / arr.length);
});
function calcVisibleCount() {
  return Math.ceil(props.AllHeight / averageHeight.value) + props.buffer;
}

/* ---------------- observers & refs ---------------- */
function initResizeObserver() {
  if (typeof ResizeObserver === "undefined") {
    ro = null;
    console.warn("ResizeObserver not available in this environment.");
    return;
  }
  ro = new ResizeObserver((entries) => {
    let needRecalc = false;
    for (const entry of entries) {
      const el = entry.target as HTMLElement;
      const idxStr = el.dataset.index;
      if (!idxStr) continue;
      const idx = Number(idxStr);
      const h = entry.contentRect.height;
      if (itemHeights.value[idx] !== h) {
        itemHeights.value[idx] = h;
        needRecalc = true;
      }
    }
    if (needRecalc) {
      // 更新 offsets 与 totalHeight，并基于当前 scrollTop 调整 start/end
      updateOffsets();
      // 保证 startIndex 在正确位置（数据可能已移动）
      startIndex.value = findStartIndexByScrollTop(scrollTop.value);
      endIndex.value = Math.min(n.value, startIndex.value + calcVisibleCount());
    }
  });
}

function setItemRef(el: HTMLElement | null, idx: number) {
  if (el) {
    el.dataset.index = String(idx);
    itemElements.set(idx, el);

    // observe newly rendered element
    if (ro && !observed.has(idx)) {
      try {
        ro.observe(el);
        observed.set(idx, el);
      } catch {}
    }
  } else {
    // removed
    itemElements.delete(idx);
    const observedEl = observed.get(idx);
    if (ro && observedEl) {
      try {
        ro.unobserve(observedEl);
      } catch {}
      observed.delete(idx);
    }
  }
}

/* 清理已不在可见区的 observed 元素（避免过多 observe）*/
function syncObserved() {
  if (!ro) return;
  const visibleSet = new Set(visibleIndices.value);
  for (const [idx, el] of Array.from(observed.entries())) {
    if (!visibleSet.has(idx)) {
      try {
        ro.unobserve(el);
      } catch {}
      observed.delete(idx);
    }
  }
  // observe current visible itemElements
  for (const idx of visibleIndices.value) {
    const el = itemElements.get(idx);
    if (el && !observed.has(idx)) {
      try {
        ro.observe(el);
        observed.set(idx, el);
      } catch {}
    }
  }
}

/* ---------------- scroll & lifecycle ---------------- */
let rafId: number | null = null;
function onScroll() {
  if (rafId != null) return;
  rafId = requestAnimationFrame(() => {
    rafId = null;
    if (!containerRef.value) return;
    scrollTop.value = containerRef.value.scrollTop;
    startIndex.value = findStartIndexByScrollTop(scrollTop.value);
    endIndex.value = Math.min(n.value, startIndex.value + calcVisibleCount());
    // 渲染后同步 observe（nextTick）
    nextTick(() => {
      syncObserved();
    });
  });
}

/* watch data changes to re-init arrays */
watch(
  () => data.value.length,
  () => {
    ensureInitHeights();
    // recalc visible window
    startIndex.value = findStartIndexByScrollTop(scrollTop.value);
    endIndex.value = Math.min(n.value, startIndex.value + calcVisibleCount());
    // after DOM updated, sync observed
    nextTick(syncObserved);
  },
  { immediate: true }
);

onMounted(() => {
  ensureInitHeights();
  initResizeObserver();
  // initial window
  startIndex.value = 0;
  endIndex.value = Math.min(n.value, calcVisibleCount());
  nextTick(() => {
    // observe/rendered elements
    syncObserved();
    // also measure any already-rendered item elements once (fallback if ResizeObserver not supported)
    if (!ro) {
      for (const [idx, el] of itemElements.entries()) {
        const h = el.getBoundingClientRect().height;
        if (itemHeights.value[idx] !== h) {
          itemHeights.value[idx] = h;
        }
      }
      updateOffsets();
    }
  });
});

onBeforeUnmount(() => {
  if (ro) {
    try {
      ro.disconnect();
    } catch {}
    ro = null;
  }
  itemElements.clear();
  observed.clear();
  if (rafId != null) {
    cancelAnimationFrame(rafId);
    rafId = null;
  }
});
</script>

<style scoped>
.virtual-list-container {
  width: 100%;
  border: 1px solid #ddd;
  box-sizing: border-box;
}
.virtual-item {
  box-sizing: border-box;
  border-bottom: 1px solid #eee;
  padding: 8px;
  background: white;
}
</style>
