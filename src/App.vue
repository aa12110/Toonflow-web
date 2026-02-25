<template>
  <t-config-provider :global-config="globalConfig">
    <el-config-provider>
      <a-config-provider :theme="theme" :locale="zhCN">
        <router-view></router-view>
        <UpdateDialog />
      </a-config-provider>
    </el-config-provider>
  </t-config-provider>
</template>

<script setup lang="ts">
import zhCN from "ant-design-vue/es/locale/zh_CN";
import UpdateDialog from "@/components/update.vue";
const theme = {
  token: {
    colorPrimary: "#9810fa",
  },
};

import { merge } from "lodash-es";
import enConfig from "tdesign-vue-next/es/locale/en_US";

import { type GlobalConfigProvider } from "tdesign-vue-next";
const empty: GlobalConfigProvider = {};
const customConfig: GlobalConfigProvider = {
  calendar: {},
  table: {},
  pagination: {},
};
const globalConfig: GlobalConfigProvider = merge(empty, enConfig, customConfig);

// document.documentElement.setAttribute('theme-mode', 'dark');
// document.documentElement.setAttribute('theme-mode', 'light');
</script>

<style lang="scss">
:root {
  --mainColor: #9810fa;
  --mainColorLight: #faf5ff;
  --mainColorHover: #7c0dd4;
  --mainColorActive: #6a0bb5;
  --mainGradient: linear-gradient(135deg, #9810fa 0%, #7c3aed 100%);
  --mainGradientHover: linear-gradient(135deg, #a020fb 0%, #8b5cf6 100%);
}
</style>
