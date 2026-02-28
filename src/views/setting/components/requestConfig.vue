<template>
  <div class="request-config">
    <t-form :data="formData" labelAlign="top" :rules="formRules" @submit="handleSubmit">
      <t-form-item label="API 地址" name="baseUrl">
        <t-input v-model="formData.baseUrl" placeholder="留空则使用同源模式（当前站点）" clearable>
          <template #prefix-icon>
            <t-icon name="link" />
          </template>
        </t-input>
      </t-form-item>
      <t-form-item label="WebSocket地址" name="wsBaseUrl">
        <t-input v-model="formData.wsBaseUrl" placeholder="留空则自动推导为同源 WS 地址" clearable>
          <template #prefix-icon>
            <t-icon name="swap" />
          </template>
        </t-input>
      </t-form-item>
      <t-form-item>
        <t-space size="small">
          <t-button theme="primary" type="submit">保存</t-button>
          <t-button theme="default" @click="handleReset">重置</t-button>
        </t-space>
      </t-form-item>
    </t-form>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";
import { MessagePlugin, type FormRules } from "tdesign-vue-next";
import useSettingStore from "@/stores/setting";

interface RequestForm {
  baseUrl: string;
  wsBaseUrl: string;
}

const settingStore = useSettingStore();

const formData = ref<RequestForm>({
  baseUrl: "",
  wsBaseUrl: "",
});

const formRules: FormRules<RequestForm> = {
  baseUrl: [
    {
      validator: (value: string) => !value || /^https?:\/\/.+/.test(value),
      message: "请输入有效的 HTTP/HTTPS 地址，或留空使用同源模式",
      trigger: "blur",
    },
  ],
  wsBaseUrl: [
    {
      validator: (value: string) => !value || /^wss?:\/\/.+/.test(value),
      message: "请输入有效的 WS/WSS 地址，或留空自动推导",
      trigger: "blur",
    },
  ],
};

function loadSettings() {
  formData.value.baseUrl = settingStore.baseUrl;
  formData.value.wsBaseUrl = settingStore.wsBaseUrl;
}

function handleSubmit({ validateResult }: { validateResult: boolean }) {
  if (validateResult) {
    settingStore.baseUrl = formData.value.baseUrl;
    settingStore.wsBaseUrl = formData.value.wsBaseUrl;
    MessagePlugin.success("请求地址保存成功");
  }
}

function handleReset() {
  formData.value.baseUrl = "";
  formData.value.wsBaseUrl = "";
  settingStore.baseUrl = formData.value.baseUrl;
  settingStore.wsBaseUrl = formData.value.wsBaseUrl;
  MessagePlugin.success("已重置为同源模式");
}

onMounted(() => {
  loadSettings();
});
</script>

<style lang="scss" scoped>
.request-config {
  padding: 10px 0;
}
</style>
