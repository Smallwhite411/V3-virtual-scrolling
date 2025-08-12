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
  <div style="margin-top: 20px">自己尝试写固定高度虚拟列表</div>
  <TextVirtal :data="listData" :itemHeight="50" :AllHeight="200">
    <template #default="{ item, index }">
      <div>{{ index }},{{ item.text }}</div> </template>
  </TextVirtal>
</template>
<script setup lang="ts">
import FixedHeightVirtualScroll from "./components/FixedHeightVirtualScroll/index.vue";
import TextVirtal from "./components/TextVirtal/index.vue";
interface Item {
  id: number;
  text: string;
}

const listData: Item[] = Array.from({ length: 10000 }, (_, i) => ({
  id: i,
  text: `Item ${i}`,
}));
</script>
<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
