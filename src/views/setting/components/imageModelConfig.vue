<template>
  <section class="settingCard">
    <div class="cardHeader">
      <div class="cardHeaderLeft">
        <div class="cardIcon green">
          <i-pic theme="outline" size="20" fill="currentColor" />
        </div>
        <span class="cardTitle">图像生成模型配置</span>
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

        <div class="formItem" v-if="!['apimart', 'runninghub'].includes(modelData.manufacturer)">
          <label class="formLabel">模型名称</label>
          <a-auto-complete
            :key="modelData.manufacturer"
            v-model:value="modelData.model"
            :options="currentModelPresets"
            placeholder="请输入或选择图像生成模型名称"
            class="formInput"
            :filter-option="filterOption" />
        </div>

        <div class="formItem" v-if="modelData.manufacturer === 'openAi'">
          <label class="formLabel">BaseURL</label>
          <a-input v-model:value="modelData.baseURL" placeholder="请输入图像生成模型的BaseURL" class="formInput" />
          <span class="formHint">提示：只需填写到 /v1/ 为止，例如 https://api.openai.com/v1/</span>
        </div>

        <div class="formItem">
          <label class="formLabel">API Key</label>
          <a-input-password v-model:value="modelData.apiKey" placeholder="请输入图像生成模型的API Key" class="formInput" />
        </div>

        <div class="formItem">
          <a-button type="primary" :loading="testingImage" @click="testImageModel" class="testBtn">
            <i-check-one v-if="!testingImage" theme="outline" size="14" fill="currentColor" />
            {{ testingImage ? "正在生成测试图片..." : "检查图像生成是否可用" }}
          </a-button>
          <span class="formHint">提示：测试将生成一张"2D猫"图片，可能需要 30秒~2分钟，请耐心等待</span>
        </div>
      </div>
    </div>
  </section>

  <!-- 图像测试结果预览弹窗 -->
  <a-modal title="图像生成测试成功" v-model:open="testImageModalVisible" :footer="null" centered width="auto" class="test-image-modal">
    <div class="test-image-content">
      <p class="test-image-tip">✅ 图像模型配置正确，以下是生成的测试图片：</p>
      <a-image :src="testImageResult" :preview="{ src: testImageResult }" class="test-image-preview" />
    </div>
  </a-modal>
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
const testingImage = ref(false);
const testImageResult = ref<string>("");
const testImageModalVisible = ref(false);
const isUpdatingFromParent = ref(false);

// 图像生成模型厂商列表
const manufacturerList = [
  { label: "OpenAI请求格式", value: "openAi" },
  { label: "火山引擎", value: "volcengine" },
  { label: "可灵", value: "kling" },
  { label: "通义万相", value: "wan" },
  { label: "gemini", value: "gemini" },
  { label: "Vidu", value: "vidu" },
  { label: "APIMart", value: "apimart" },
  { label: "RunningHub", value: "runninghub" },
];

// 各厂商的模型预设
const modelPresetsByManufacturer = {
  openAi: [
    { label: "dall-e-3", value: "dall-e-3" },
    { label: "dall-e-2", value: "dall-e-2" },
  ],
  wan: [{ label: "wan2.6-image", value: "wan2.6-image" }],
  volcengine: [
    { label: "doubao-seedream-4-5", value: "doubao-seedream-4-5" },
    { label: "doubao-seedream-4-0", value: "doubao-seedream-4-0" },
  ],
  kling: [
    { label: "kling-image-o1", value: "kling-image-o1" },
    { label: "kling-v1-5", value: "kling-v1-5" },
    { label: "kling-v2", value: "kling-v2" },
    { label: "kling-v2-1", value: "kling-v2-1" },
  ],
  gemini: [
    { label: "gemini-2.5-flash-image", value: "gemini-2.5-flash-image" },
    { label: "gemini-2.5-flash-image-preview", value: "gemini-2.5-flash-image-preview" },
    { label: "gemini-2.5-flash-image-preview-all", value: "gemini-2.5-flash-image-preview-all" },
    { label: "gemini-3-pro-image-preview", value: "gemini-3-pro-image-preview" },
  ],
  vidu: [{ label: "viduq2", value: "viduq2" }],
  apimart: [{ label: "nanobanana", value: "nanobanana" }],
  runninghub: [{ label: "nanobanana", value: "nanobanana" }],
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
  // 清空当前模型名称，让用户重新填写
  modelData.value.model = "";
  emit("change");
}

// 测试图像模型连接
async function testImageModel() {
  const { model, apiKey, baseURL, manufacturer } = modelData.value;

  // APIMart 和 RunningHub 不需要模型名称
  if (!["apimart", "runninghub"].includes(manufacturer) && !model) {
    message.warning("请先填写模型名称");
    return;
  }
  if (!apiKey) {
    message.warning("请先填写 API Key");
    return;
  }

  testingImage.value = true;
  message.info("正在生成测试图片，请耐心等待...");
  try {
    const res = await axios.post("/other/testImage", {
      modelName: model || undefined,
      apiKey: apiKey,
      baseURL: baseURL || undefined,
      manufacturer: manufacturer,
    });
    message.success("连接成功！图像模型配置正确");
    // 显示生成的图片
    if (res.data) {
      testImageResult.value = res.data;
      testImageModalVisible.value = true;
    }
  } catch (e: any) {
    message.error(`连接失败: ${e.message}`);
  } finally {
    testingImage.value = false;
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

.cardIcon.green {
  background: linear-gradient(135deg, #dcfce7 0%, #bbf7d0 100%);
  color: #22c55e;
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

/* AutoComplete 样式 */
:deep(.ant-select-auto-complete .ant-select-selector) {
  height: 44px !important;
  border-radius: 10px !important;
  border: 1px solid #e5e7eb !important;
  background: #f9fafb !important;
  padding: 0 16px !important;
}

:deep(.ant-select-auto-complete .ant-select-selection-search-input) {
  height: 42px !important;
}

:deep(.ant-select-auto-complete:hover .ant-select-selector) {
  border-color: #d1d5db !important;
}

:deep(.ant-select-auto-complete.ant-select-focused .ant-select-selector) {
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

/* 图像测试结果弹窗 */
.test-image-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
  padding: 8px;
}

.test-image-tip {
  color: #22c55e;
  font-size: 14px;
  font-weight: 500;
  margin: 0;
}

.test-image-preview {
  max-width: 512px;
  max-height: 512px;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}
</style>
