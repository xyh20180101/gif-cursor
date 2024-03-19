<script setup>
import { computed } from "vue"
import { useOsTheme, darkTheme, NConfigProvider, NTabs, NTabPane } from 'naive-ui'
import SingleCursorPage from './components/SingleCursorPage.vue'
import MultiCursorPage from './components/MultiCursorPage.vue'
</script>

<template>
  <n-config-provider :theme="theme">
    <span style="display: flex;justify-content: center;margin-bottom: 32px;">将gif动态图片转为纯css/js，而无需引入npm包</span>
    <n-tabs justify-content="space-evenly" animated>
      <n-tab-pane name="chap1" display-directive="show" tab="单一指针（全局）">
        <SingleCursorPage />
      </n-tab-pane>
      <n-tab-pane name="chap2" display-directive="show" tab="多类型指针">
        <MultiCursorPage />
      </n-tab-pane>
    </n-tabs>
  </n-config-provider>
</template>

<script>
export default {
  data() {
    return {
      theme: computed(() => useOsTheme().value === "dark" ? darkTheme : null)
    }
  }
}
</script>

<style>
.n-tab-pane{
  padding-top: 16px !important;
}
</style>