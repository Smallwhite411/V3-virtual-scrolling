<template>
  <img alt="Vue logo" src="./assets/logo.png" />
  <div>固定高度的虚拟列表</div>
  <FixedHeightVirtualScroll
    :items="listData"
    :itemHeight="50"
    :containerHeight="200"
    keyField="id"
    :buffer="5"
  >
    <template #default="{ item, index }">
      <div>{{ index }} - {{ item.text }}</div>
    </template>
  </FixedHeightVirtualScroll>
  <div class="title-style">优化固定高度的虚拟列表</div>
  <OptFixedHeightVirtualScroll
    ref="listRef"
    :items="listData"
    :itemHeight="50"
    :height="300"
  >
    <template #default="{ item, index }">
      <div style="padding-left: 10px; flex: 1">{{ index }} - {{ item }}</div>
    </template>
  </OptFixedHeightVirtualScroll>
  <button @click="scrollTo100">滚动到第100项</button>
  <div class="title-style">自己尝试写固定高度虚拟列表</div>
  <TextVirtal :data="listData" :itemHeight="50" :AllHeight="200">
    <template #default="{ item, index }">
      <div>{{ index }},{{ item.text }}</div>
    </template>
  </TextVirtal>
  <div class="title-style">
    自己尝试写不固定高度虚拟列表(不固定高度的高度指的是item的高度)
  </div>
  <TextVaribleHeightVirtual
    :allItems="allItems"
    :itemHeight="50"
    :AllHeight="200"
    keyField="id"
  >
    <template #default="{ item }">
      <div
        :style="{
          display: 'flex',
          alignItems: 'center',
        }"
      >
        {{ item.text }}
      </div>
    </template>
  </TextVaribleHeightVirtual>
  <div class="title-style">可变高度 + 无限滚动虚拟列表</div>
  <VariableHeightVirtual
    :data="items"
    :AllHeight="300"
    :buffer="5"
    keyField="id"
  >
    <template #default="{ item, index }">
      <div
        :data-index="index"
        :style="{
          padding: '10px',
          borderBottom: '1px solid #ccc',
        }"
      >
        {{ index }} - {{ item }}
      </div>
    </template>
  </VariableHeightVirtual>
</template>
<script setup lang="ts">
import FixedHeightVirtualScroll from "./components/FixedHeightVirtualScroll/index.vue";
import TextVirtal from "./components/TextVirtal/index.vue";
import OptFixedHeightVirtualScroll from "./components/OptFixedHeightVirtual/index.vue";
import VariableHeightVirtual from "./components/VariableHeightVirtual/index.vue";
import TextVaribleHeightVirtual from "./components/TextVaribleHeightVirtual/index.vue";
import { ref } from "vue";
export interface Item {
  id: number;
  text: string;
  height: number;
}

const listData: Item[] = Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  text: `Item ${i}`,
  height: Math.floor(Math.random() * 100) + 50,
}));

const listRef = ref<InstanceType<typeof OptFixedHeightVirtualScroll> | null>(
  null
);
const scrollTo100 = () => {
  listRef.value?.scrollToIndex(100);
};
const items = ref(
  Array.from({ length: 50 }, (_, i) => ({
    id: i,
    text: "Item " + i + " " + "x".repeat(Math.floor(Math.random() * 20)),
  }))
);
const allItems = Array.from({ length: 100 }, (v, i) => {
  return {
    id: i,
    text: "Itemdddd " + i + " " + "x".repeat(Math.floor(Math.random() * 20)),
  };
});
</script>
<style lang="scss">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
.title-style {
  margin-top: 20px;
}
</style>
