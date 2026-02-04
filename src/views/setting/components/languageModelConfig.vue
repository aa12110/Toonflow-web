<template>
  <section class="settingCard">
    <div class="cardHeader">
      <div class="cardHeaderLeft">
        <div class="cardIcon purple">
          <i-brain theme="outline" size="20" fill="currentColor" />
        </div>
        <span class="cardTitle">语言生成模型配置</span>
      </div>
    </div>

    <div class="cardContent">
      <div class="modelForm">
        <div class="formItem">
          <label class="formLabel">厂商</label>
          <a-select v-model:value="modelData.manufacturer" placeholder="请选择厂商" class="formSelect" @change="handleManufacturerChange">
            <a-select-option v-for="item in manufacturerList" :key="item.value" :value="item.value">
              {{ item.label }}
            </a-select-option>
          </a-select>
        </div>
        <div class="formItem">
          <label class="formLabel">模型名称</label>
          <a-auto-complete
            :key="modelData.manufacturer"
            v-model:value="modelData.model"
            :options="currentModelPresets"
            placeholder="请输入或选择语言生成模型名称"
            class="formInput"
            :filter-option="filterOption" />
        </div>
        <div class="formItem">
          <label class="formLabel">BaseURL</label>
          <a-input
            v-model:value="modelData.baseURL"
            placeholder="请输入语言生成模型的BaseURL"
            class="formInput"
            :class="{ 'input-error': baseURLError }"
            @input="validateBaseURL" />
          <span class="formError" v-if="baseURLError">{{ baseURLError }}</span>
          <span class="formHint" v-else>提示：只需填写到 /v1/ 为止，例如 https://api.openai.com/v1/</span>
        </div>
        <div class="formItem">
          <label class="formLabel">API Key</label>
          <a-input-password v-model:value="modelData.apiKey" placeholder="请输入语言生成模型的API Key" class="formInput" />
        </div>
        <div class="formItem">
          <a-button type="primary" :loading="testingAI" @click="testLanguageModel" class="testBtn">
            <i-check-one v-if="!testingAI" theme="outline" size="14" fill="currentColor" />
            检查AI是否可用
          </a-button>
        </div>
      </div>
    </div>
  </section>
</template>

<script setup lang="ts">
import { ref, watch, computed } from "vue";
import { message } from "ant-design-vue";
import axios from "@/utils/axios";

interface ModelDataType {
  model: string;
  apiKey: string;
  baseURL: string;
  manufacturer: string;
}

const emit = defineEmits<{
  (e: "update:modelValue", value: ModelDataType): void;
  (e: "change"): void;
}>();

const modelData = defineModel<ModelDataType>({
  default: {
    model: "",
    apiKey: "",
    baseURL: "",
    manufacturer: "",
  },
});
const testingAI = ref(false);
const baseURLError = ref("");

// 语言生成模型厂商列表
const manufacturerList = [
  { label: "OpenAI请求格式", value: "openAi", baseUrl: "" },
  { label: "DeepSeek", value: "deepseek", baseUrl: "https://api.deepseek.com/v1/" },
  { label: "火山引擎", value: "volcengine", baseUrl: "https://ark.cn-beijing.volces.com/api/v3/" },
  { label: "通义千问", value: "qwen", baseUrl: "https://dashscope.aliyuncs.com/compatible-mode/v1/" },
  { label: "智谱", value: "zhipu", baseUrl: "https://open.bigmodel.cn/api/paas/v4/" },
  { label: "谷歌", value: "google", baseUrl: "" },
  { label: "Anthropic", value: "anthropic", baseUrl: "" },
];

// 各厂商的模型预设
const modelPresetsByManufacturer = {
  openAi: [
    { label: "gpt-4o", value: "gpt-4o" },
    { label: "gpt-4o-mini", value: "gpt-4o-mini" },
    { label: "gpt-4.1", value: "gpt-4.1" },
    { label: "gpt-5.1", value: "gpt-5.1" },
    { label: "gpt-5.2", value: "gpt-5.2" },
  ],
  anthropic: [
    { label: "claude-opus-4-5", value: "claude-opus-4-5" },
    { label: "claude-haiku-4-5", value: "claude-haiku-4-5" },
    { label: "claude-sonnet-4-5", value: "claude-sonnet-4-5" },
    { label: "claude-opus-4-1", value: "claude-opus-4-1" },
    { label: "claude-opus-4-0", value: "claude-opus-4-0" },
    { label: "claude-sonnet-4-0", value: "claude-sonnet-4-0" },
    { label: "claude-3-7-sonnet-latest", value: "claude-3-7-sonnet-latest" },
    { label: "claude-3-5-haiku-latest", value: "claude-3-5-haiku-latest" },
  ],
  google: [
    { label: "gemini-2.5-pro", value: "gemini-2.5-pro" },
    { label: "gemini-2.5-flash", value: "gemini-2.5-flash" },
    { label: "gemini-2.0-flash", value: "gemini-2.0-flash" },
    { label: "gemini-2.0-flash-lite", value: "gemini-2.0-flash-lite" },
    { label: "gemini-1.5-pro", value: "gemini-1.5-pro" },
    { label: "gemini-1.5-flash", value: "gemini-1.5-flash" },
  ],
  zhipu: [
    { label: "glm-4.7", value: "glm-4.7" },
    { label: "glm-4.7-flashx", value: "glm-4.7-flashx" },
    { label: "glm-4.6", value: "glm-4.6" },
    { label: "glm-4.5-air", value: "glm-4.5-air" },
    { label: "glm-4.5-airx", value: "glm-4.5-airx" },
    { label: "glm-4-long", value: "glm-4-long" },
    { label: "glm-4-flashx-250414", value: "glm-4-flashx-250414" },
    { label: "glm-4.7-flash", value: "glm-4.7-flash" },
    { label: "glm-4.5-flash", value: "glm-4.5-flash" },
    { label: "glm-4-flash-250414", value: "glm-4-flash-250414" },
    { label: "glm-4.6v", value: "glm-4.6v" },
  ],
  qwen: [
    { label: "qwen-vl-max", value: "qwen-vl-max" },
    { label: "qwen-plus-latest", value: "qwen-plus-latest" },
    { label: "qwen-max", value: "qwen-max" },
    { label: "qwen2.5-72b-instruct", value: "qwen2.5-72b-instruct" },
    { label: "qwen2.5-14b-instruct-1m", value: "qwen2.5-14b-instruct-1m" },
    { label: "qwen2.5-vl-72b-instruct", value: "qwen2.5-vl-72b-instruct" },
  ],
  deepseek: [
    { label: "deepseek-chat", value: "deepseek-chat" },
    { label: "deepseek-reasoner", value: "deepseek-reasoner" },
  ],
  volcengine: [
    { label: "doubao-seed-1-6-flash", value: "doubao-seed-1-6-flash" },
    { label: "doubao-seed-1-6-lite", value: "doubao-seed-1-6-lite-251015" },
    { label: "doubao-seed-1-6", value: "doubao-seed-1-6" },
    { label: "doubao-seed-1-8", value: "doubao-seed-1-8" },
  ],
};

// 当前选中厂商的模型预设
const currentModelPresets = computed(() => {
  const manufacturer = modelData.value.manufacturer || "openAi";

  return modelPresetsByManufacturer[manufacturer as keyof typeof modelPresetsByManufacturer] || modelPresetsByManufacturer.openAi;
});

// AutoComplete 过滤选项
function filterOption(input: string, option: any) {
  return option.value.toLowerCase().includes(input.toLowerCase());
}

function handleManufacturerChange() {
  // 根据选择的厂商自动填充 baseUrl
  const manufacturer = manufacturerList.find((item) => item.value === modelData.value.manufacturer);
  if (manufacturer && manufacturer.baseUrl) {
    modelData.value.baseURL = manufacturer.baseUrl;
  } else {
    modelData.value.baseURL = "";
  }
  // 清空当前模型名称，让用户重新填写
  modelData.value.model = "";

  emit("change");
}

// 校验 BaseURL 格式
function validateBaseURL() {
  const url = modelData.value.baseURL;
  if (!url) {
    baseURLError.value = "";
    return true;
  }

  // 检查是否包含 /v1/ 后面还有其他路径
  const v1Index = url.indexOf("/v1/");
  if (v1Index !== -1) {
    const afterV1 = url.substring(v1Index + 4); // 获取 /v1/ 后面的内容
    if (afterV1 && afterV1 !== "/" && afterV1.replace(/\/+$/, "") !== "") {
      baseURLError.value = "错误：BaseURL 只需填写到 /v1/ 为止，请勿添加后续路径";
      return false;
    }
  }

  baseURLError.value = "";
  return true;
}

// 测试语言模型连接
async function testLanguageModel() {
  const { model, apiKey, baseURL } = modelData.value;

  if (!model) {
    message.warning("请先填写模型名称");
    return;
  }
  if (!apiKey) {
    message.warning("请先填写 API Key");
    return;
  }

  testingAI.value = true;
  try {
    await axios.post("/other/testAI", {
      modelName: model,
      apiKey: apiKey,
      baseURL: baseURL || undefined,
    });
    message.success("连接成功！模型配置正确");
  } catch (e: any) {
    message.error(`连接失败: ${e.message}`);
  } finally {
    testingAI.value = false;
  }
}
</script>

<style scoped>
/* 卡片通用样式 */
.settingCard {
  background: #fff;
  border-radius: 16px;
  box-shadow:
    0 1px 3px rgba(0, 0, 0, 0.05),
    0 4px 12px rgba(152, 16, 250, 0.04);
  border: 1px solid #f3e8ff;
  margin-bottom: 20px;
  overflow: hidden;
  transition: all 0.2s;
}

.settingCard:hover {
  box-shadow: 0 4px 16px rgba(152, 16, 250, 0.08);
}

.cardHeader {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 20px 24px;
  border-bottom: 1px solid #f9fafb;
}

.cardHeaderLeft {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1;
}

.cardIcon {
  width: 40px;
  height: 40px;
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.cardIcon.purple {
  background: linear-gradient(135deg, #f3e8ff 0%, #e9d5ff 100%);
  color: var(--mainColor);
}

.cardTitle {
  font-size: 16px;
  font-weight: 600;
  color: #1f2937;
}

.cardContent {
  padding: 24px;
}

/* 表单样式 */
.formItem {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.modelForm .formItem + .formItem {
  margin-top: 16px;
}

.formLabel {
  font-size: 13px;
  font-weight: 500;
  color: #4b5563;
}

.formHint {
  font-size: 12px;
  color: #9ca3af;
  margin-top: 4px;
}

.formError {
  font-size: 12px;
  color: #ef4444;
  margin-top: 4px;
}

.input-error {
  border-color: #ef4444 !important;
}

.input-error:focus,
.input-error:focus-within {
  border-color: #ef4444 !important;
  box-shadow: 0 0 0 3px rgba(239, 68, 68, 0.1) !important;
}

.formInput {
  height: 44px;
  border-radius: 10px;
  border: 1px solid #e5e7eb;
  background: #f9fafb;
  transition: all 0.2s;
}

.formInput:hover {
  border-color: #d1d5db;
}

.formInput:focus,
.formInput:focus-within {
  border-color: var(--mainColor);
  background: #fff;
  box-shadow: 0 0 0 3px rgba(152, 16, 250, 0.1);
}

.formSelect {
  width: 100%;
}

:deep(.ant-select-selector) {
  height: 44px !important;
  border-radius: 10px !important;
  border: 1px solid #e5e7eb !important;
  background: #f9fafb !important;
  padding: 0 16px !important;
}

:deep(.ant-select-selection-item) {
  line-height: 42px !important;
}

:deep(.ant-select-selection-placeholder) {
  line-height: 42px !important;
}

:deep(.ant-select:hover .ant-select-selector) {
  border-color: #d1d5db !important;
}

:deep(.ant-select-focused .ant-select-selector) {
  border-color: var(--mainColor) !important;
  background: #fff !important;
  box-shadow: 0 0 0 3px rgba(152, 16, 250, 0.1) !important;
}

.testBtn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  width: fit-content;
}

/* 主题色按钮覆盖 */
:deep(.ant-btn-primary) {
  background: var(--mainGradient);
  border: none;
}

:deep(.ant-btn-primary:hover) {
  background: var(--mainGradientHover);
}
</style>
