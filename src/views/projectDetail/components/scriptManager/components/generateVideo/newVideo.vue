<template>
  <div class="newStoryboard">
    <a-modal
      width="80vw"
      :style="{ top: '20px' }"
      v-model:open="storyboardShow"
      title="添加视频配置"
      okText="保存配置"
      @ok="handleOk"
      @cancel="handleCancel"
      :confirmLoading="generateVideoLoading">
      <div class="configPanel">
        <div class="configHeader">
          <h3>视频生成配置</h3>
          <a-button type="primary" @click="addVideoConfig">
            <plus-outlined />
            添加配置
          </a-button>
        </div>
        <!-- 视频配置列表 Grid 布局 -->
        <div class="configList" v-if="videoConfigs.length > 0">
          <div v-for="(config, index) in videoConfigs" :key="config.id" class="configCard">
            <div class="cardHeader">
              <span class="cardTitle">配置 {{ index + 1 }}</span>
              <a-button type="text" danger size="small" @click="removeVideoConfig(index)">
                <delete-outlined />
              </a-button>
            </div>
            <div class="cardBody">
              <!-- 厂商选择 -->
              <div class="formRow">
                <label>模型</label>
                <a-select v-model:value="config.configId" @change="onManufacturerChange(config)" :disabled="allManufacturerDisable" size="small">
                  <a-select-option v-for="item in availableManufacturers" :key="item.value" :value="item.value">
                    {{ item.label }}
                  </a-select-option>
                </a-select>
              </div>
              <!-- 模式选择 -->
              <div class="formRow">
                <label>模式</label>
                <a-radio-group v-model:value="config.mode" @change="onModeChange(config)" size="small">
                  <a-radio v-for="mode in getModeOptions(config.manufacturer, config.model)" :key="mode.value" :value="mode.value">
                    {{ mode.label }}
                  </a-radio>
                </a-radio-group>
              </div>
              <!-- 首尾帧模式配置 -->
              <template v-if="config.mode === 'startEnd'">
                <div class="formRow frameRow">
                  <label>帧选择</label>
                  <div class="frameGroup">
                    <div class="frameBox" :class="{ hasImage: config.startFrame }" @click="openImageSelector(config, 'start')">
                      <template v-if="config.startFrame">
                        <img :src="config.startFrame.filePath" />
                        <a-button class="removeBtn" type="text" size="small" @click.stop="config.startFrame = null">
                          <close-outlined />
                        </a-button>
                        <span class="frameLabel">首帧</span>
                      </template>
                      <template v-else>
                        <plus-outlined />
                        <span>首帧</span>
                      </template>
                    </div>
                    <div class="frameBox" :class="{ hasImage: config.endFrame }" @click="openImageSelector(config, 'end')">
                      <template v-if="config.endFrame">
                        <img :src="config.endFrame.filePath" />
                        <a-button class="removeBtn" type="text" size="small" @click.stop="config.endFrame = null">
                          <close-outlined />
                        </a-button>
                        <span class="frameLabel">尾帧</span>
                      </template>
                      <template v-else>
                        <plus-outlined />
                        <span>尾帧</span>
                      </template>
                    </div>
                  </div>
                </div>
              </template>
              <!-- 多图模式配置 -->
              <template v-if="config.mode === 'multi'">
                <div class="formRow">
                  <label>图片</label>
                  <div class="multiImages">
                    <draggable
                      v-if="config.images && config.images.length > 0"
                      v-model="config.images"
                      item-key="id"
                      class="imageDragList"
                      ghost-class="ghost"
                      :animation="200"
                      handle=".dragHandle">
                      <template #item="{ element, index: imgIndex }">
                        <div class="dragImageItem">
                          <div class="dragHandle">
                            <img class="image" :src="element.filePath" draggable="false" />
                            <div class="imageOrder">{{ imgIndex + 1 }}</div>
                          </div>
                          <a-button class="removeBtn" type="text" size="small" @click="removeImageFromConfig(config, imgIndex)">
                            <close-outlined />
                          </a-button>
                        </div>
                      </template>
                    </draggable>
                    <div
                      class="addImageBox"
                      @click="openImageSelector(config, 'multi')"
                      v-if="!config.images || config.images.length < getMaxImages(config.manufacturer, config.model)">
                      <plus-outlined />
                    </div>
                  </div>
                </div>
                <div class="formRow">
                  <label></label>
                  <span class="tip">拖拽调整顺序 | {{ config.images?.length || 0 }}/{{ getMaxImages(config.manufacturer, config.model) }}张</span>
                </div>
              </template>
              <!-- 单图模式配置 -->
              <template v-if="config.mode === 'single'">
                <div class="formRow frameRow">
                  <label>图片</label>
                  <div class="frameGroup">
                    <div class="frameBox singleFrame" :class="{ hasImage: config.startFrame }" @click="openImageSelector(config, 'start')">
                      <template v-if="config.startFrame">
                        <img :src="config.startFrame.filePath" />
                        <a-button class="removeBtn" type="text" size="small" @click.stop="config.startFrame = null">
                          <close-outlined />
                        </a-button>
                      </template>
                      <template v-else>
                        <plus-outlined />
                        <span>选择图片</span>
                      </template>
                    </div>
                  </div>
                </div>
              </template>
              <!-- 分辨率/比例 -->
              <div class="formRow">
                <label>{{ getResolutionLabel(config.manufacturer, config.model) }}</label>
                <a-select v-model:value="config.resolution" size="small" style="width: 140px">
                  <a-select-option v-for="res in getResolutionOptions(config.manufacturer, config.model)" :key="res.value" :value="res.value">
                    {{ res.label }}
                  </a-select-option>
                </a-select>
              </div>
              <!-- 时长 -->
              <div class="formRow">
                <label>时长</label>
                <template v-if="getDurationOptions(config.manufacturer, config.model).length > 0">
                  <a-select v-model:value="config.duration" size="small" style="width: 100px">
                    <a-select-option v-for="dur in getDurationOptions(config.manufacturer, config.model)" :key="dur.value" :value="dur.value">
                      {{ dur.label }}
                    </a-select-option>
                  </a-select>
                </template>
                <template v-else>
                  <a-input-number
                    v-model:value="config.duration"
                    :min="getDurationRange(config.manufacturer, config.model).min"
                    :max="getDurationRange(config.manufacturer, config.model).max"
                    :step="getDurationRange(config.manufacturer, config.model).step"
                    size="small"
                    style="width: 70px" />
                  <span class="unit">秒</span>
                  <span class="tip">{{ getDurationTip(config.manufacturer, config.model) }}</span>
                </template>
              </div>
              <!-- 视频提示词 -->
              <div class="formRow promptRow">
                <label>提示词</label>
                <div class="promptWrapper">
                  <a-textarea v-model:value="config.prompt" :rows="2" placeholder="描述视频内容、运动方式等" size="small" />
                  <a-button
                    class="magicBtn"
                    type="link"
                    size="small"
                    :loading="config.promptLoading"
                    @click="generateConfigPrompt(config)"
                    style="margin-right: 20px">
                    <i-magic />
                  </a-button>
                </div>
              </div>
            </div>
          </div>
        </div>
        <a-empty v-else description="请点击上方按钮添加视频配置" />
      </div>
    </a-modal>
    <!-- 分镜图片选择弹窗 -->
    <a-modal
      v-model:open="imageSelectorVisible"
      :title="imageSelectorTitle"
      @ok="confirmImageSelection"
      @cancel="imageSelectorVisible = false"
      width="80%"
      :bodyStyle="{ maxHeight: '70vh', overflow: 'auto' }">
      <mainElement
        v-if="imageSelectorVisible"
        :way="imageSelectorMode === 'multi' ? 'checkbox' : 'radio'"
        radio="storyboard"
        ref="mainElementRef"
        @checkChange="handleCheckedChange"
        @check-all="handleBatchCheckAll" />
      <template #footer>
        <div class="selectorFooter">
          <span class="selectedCount">已选择 {{ tempSelectedImages.length }} 张</span>
          <div>
            <a-button @click="imageSelectorVisible = false">取消</a-button>
            <a-button type="primary" @click="confirmImageSelection">确定</a-button>
          </div>
        </div>
      </template>
    </a-modal>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, onMounted, computed } from "vue";
import { message } from "ant-design-vue";
import { PlusOutlined, DeleteOutlined, CloseOutlined } from "@ant-design/icons-vue";
import draggable from "vuedraggable";
import mainElement from "@/views/projectDetail/components/assetsManager/components/mainElement.vue";
import axios from "@/utils/axios";
import store from "@/stores";
import videoStore from "@/stores/video";
import {
  type ImageItem,
  manufacturerConfigs,
  getManufacturerConfig,
  getModeOptions,
  getResolutionLabel,
  getResolutionOptions,
  getDefaultResolution,
  getDefaultMode,
  getDefaultDuration,
  getDurationOptions,
  getDurationRange,
  getDurationTip,
  getMaxImages,
} from "@/components/videoConfig";

const storeInstance = store();
const { project, currentScriptId } = storeToRefs(storeInstance);
const videoStoreInstance = videoStore();

interface VideoConfig {
  id: number;
  manufacturer: string;
  model: string;
  configId: number | undefined;
  mode: "startEnd" | "multi" | "single";
  startFrame: ImageItem | null;
  endFrame: ImageItem | null;
  images: ImageItem[];
  resolution: string;
  duration: number;
  prompt: string;
  promptLoading: boolean;
}
interface Storyboard {
  id: number;
  name: string;
  filePath: string;
  prompt: string;
  duration: number;
}
const props = defineProps<{ scriptId: number }>();
const storyboardShow = defineModel<boolean>({});
const generateVideoLoading = ref(false);
const allManufacturerDisable = ref(false);
const videoConfigs = ref<VideoConfig[]>([]);
let configIdCounter = 0;
const imageSelectorVisible = ref(false);
const imageSelectorTitle = ref("选择分镜图片");
const imageSelectorMode = ref<"start" | "end" | "multi">("start");
const currentEditConfig = ref<VideoConfig | null>(null);
const tempSelectedImages = ref<ImageItem[]>([]);
const tempSelectedIds = ref<number[]>([]);
const manufacturerList = ref<{ model: string; manufacturer: string; id: number }[]>([]);
const manufacturerAllRecord: Record<string, string> = Object.values(manufacturerConfigs).reduce((acc: Record<string, string>, c) => {
  acc[c.value as string] = c.label;
  return acc;
}, {});
const availableManufacturers = computed(() => {
  if (manufacturerList.value.length === 0) return [];
  return manufacturerList.value.map((i) => ({ label: i.model + manufacturerAllRecord[i.manufacturer], value: i.id, manufacturer: i.manufacturer }));
});
onMounted(async () => {
  const res = await axios.post("/video/getManufacturer", {
    userId: Number(localStorage.getItem("userId")),
  });
  manufacturerList.value = res.data;
  console.log("%c Line:295 🍇 manufacturerList.value", "background:#6ec1c2", manufacturerList.value);
  allManufacturerDisable.value = manufacturerList.value.length === 0;
});
watch(storyboardShow, (v) => {
  if (v) {
    videoConfigs.value = [];
  }
});

function addVideoConfig() {
  const defaultManufacturer: string = availableManufacturers.value[0]?.manufacturer || "volcengine";
  const defaultModel: string = availableManufacturers.value[0] ? manufacturerList.value.find(i => i.id === availableManufacturers.value[0].value)?.model || "" : "";
  const newConfig: VideoConfig = {
    id: ++configIdCounter,
    configId: undefined,
    manufacturer: "",
    model: "",
    mode: getDefaultMode(defaultManufacturer, defaultModel) as VideoConfig["mode"],
    startFrame: null,
    endFrame: null,
    images: [],
    resolution: getDefaultResolution(defaultManufacturer, defaultModel),
    duration: getDefaultDuration(defaultManufacturer, defaultModel),
    prompt: "",
    promptLoading: false,
  };
  videoConfigs.value.push(newConfig);
}
function removeVideoConfig(index: number) {
  videoConfigs.value.splice(index, 1);
}
function onManufacturerChange(config: VideoConfig) {
  const selectedItem = manufacturerList.value.find((i) => i.id == config.configId);
  config.manufacturer = selectedItem?.manufacturer!;
  config.model = selectedItem?.model || "";
  const manufacturerConfig = getManufacturerConfig(config.manufacturer, config.model);
  // 重置模式
  config.mode = manufacturerConfig.defaultMode as VideoConfig["mode"];
  config.resolution = manufacturerConfig.defaultResolution;
  config.duration = getDefaultDuration(config.manufacturer, config.model);
  // 清空图片选择
  config.startFrame = null;
  config.endFrame = null;
  config.images = [];
}
function onModeChange(config: VideoConfig) {
  config.startFrame = null;
  config.endFrame = null;
  config.images = [];
}
function openImageSelector(config: VideoConfig, type: "start" | "end" | "multi") {
  // 同步 scriptId 到 store，确保 mainElement 能获取正确的分镜数据
  if (props.scriptId && props.scriptId !== -1) {
    currentScriptId.value = props.scriptId;
  }
  currentEditConfig.value = config;
  imageSelectorMode.value = type;
  const titleMap: Record<string, string> = {
    start: config.mode === "single" ? "选择图片" : "选择首帧图片",
    end: "选择尾帧图片",
    multi: "选择图片序列（可多选）",
  };
  imageSelectorTitle.value = titleMap[type];
  if (type === "start" && config.startFrame) {
    tempSelectedImages.value = [config.startFrame];
    tempSelectedIds.value = [config.startFrame.id];
  } else if (type === "end" && config.endFrame) {
    tempSelectedImages.value = [config.endFrame];
    tempSelectedIds.value = [config.endFrame.id];
  } else if (type === "multi") {
    tempSelectedImages.value = [...config.images];
    tempSelectedIds.value = config.images.map((img) => img.id);
  } else {
    tempSelectedImages.value = [];
    tempSelectedIds.value = [];
  }
  imageSelectorVisible.value = true;
}
function handleBatchCheckAll(data: { checked: boolean; records: Storyboard[] }, type: string) {
  if (type !== "storyboard") return;
  const isSingleSelect = imageSelectorMode.value !== "multi";
  const maxImages = getMaxImages(currentEditConfig.value?.manufacturer || "", currentEditConfig.value?.model || "");
  if (data.checked) {
    if (isSingleSelect) {
      if (data.records.length > 0) {
        const row = data.records[0];
        tempSelectedIds.value = [row.id];
        tempSelectedImages.value = [{ id: row.id, filePath: row.filePath, prompt: row.prompt }];
      }
    } else {
      data.records.forEach((row) => {
        if (!tempSelectedIds.value.includes(row.id) && tempSelectedImages.value.length < maxImages) {
          tempSelectedIds.value.push(row.id);
          tempSelectedImages.value.push({ id: row.id, filePath: row.filePath, prompt: row.prompt });
        }
      });
    }
  } else {
    data.records.forEach((row) => {
      const index = tempSelectedIds.value.indexOf(row.id);
      if (index > -1) {
        tempSelectedIds.value.splice(index, 1);
        tempSelectedImages.value.splice(index, 1);
      }
    });
  }
}
function handleCheckedChange(data: { checked: boolean; row: Storyboard }) {
  const isSingleSelect = imageSelectorMode.value !== "multi";
  const maxImages = getMaxImages(currentEditConfig.value?.manufacturer || "", currentEditConfig.value?.model || "");
  if (data.checked) {
    if (isSingleSelect) {
      tempSelectedIds.value = [data.row.id];
      tempSelectedImages.value = [{ id: data.row.id, filePath: data.row.filePath, prompt: data.row.prompt }];
    } else {
      if (!tempSelectedIds.value.includes(data.row.id)) {
        if (tempSelectedImages.value.length >= maxImages) {
          message.warning(`最多只能选择${maxImages}张图片`);
          return;
        }
        tempSelectedIds.value.push(data.row.id);
        tempSelectedImages.value.push({ id: data.row.id, filePath: data.row.filePath, prompt: data.row.prompt });
      }
    }
  } else {
    const index = tempSelectedIds.value.indexOf(data.row.id);
    if (index > -1) {
      tempSelectedIds.value.splice(index, 1);
      tempSelectedImages.value.splice(index, 1);
    }
  }
}
function confirmImageSelection() {
  if (!currentEditConfig.value) return;
  if (imageSelectorMode.value === "start") {
    currentEditConfig.value.startFrame = tempSelectedImages.value[0] || null;
  } else if (imageSelectorMode.value === "end") {
    currentEditConfig.value.endFrame = tempSelectedImages.value[0] || null;
  } else {
    currentEditConfig.value.images = [...tempSelectedImages.value];
  }
  imageSelectorVisible.value = false;
}
function removeImageFromConfig(config: VideoConfig, index: number) {
  config.images.splice(index, 1);
}
async function generateConfigPrompt(config: VideoConfig) {
  const images: ImageItem[] = [];
  if (config.mode === "startEnd") {
    if (config.startFrame) images.push(config.startFrame);
    if (config.endFrame) images.push(config.endFrame);
  } else if (config.mode === "single") {
    if (config.startFrame) images.push(config.startFrame);
  } else {
    images.push(...config.images);
  }
  if (images.length === 0) {
    message.warning("请先选择图片");
    return;
  }
  config.promptLoading = true;
  try {
    const res = await axios.post("/video/generatePrompt", {
      prompt: config.prompt || "生成视频",
      images: images.map((img) => ({ filePath: img.filePath, prompt: img.prompt })),
      duration: config.duration,
      type: config.mode,
    });
    config.prompt = res.data;
    message.success("提示词生成成功");
  } catch (e: any) {
    console.log("%c Line:463 🌽 e", "background:#e41a6a", e);
    message.error(e?.message || "生成失败");
  } finally {
    config.promptLoading = false;
  }
}
async function handleOk() {
  if (videoConfigs.value.length === 0) {
    message.warning("请至少添加一个视频配置");
    return;
  }
  for (let i = 0; i < videoConfigs.value.length; i++) {
    const config = videoConfigs.value[i];
    if (config.mode === "startEnd" && !config.startFrame) {
      message.warning(`配置${i + 1}：请选择首帧图片`);
      return;
    }
    if (config.mode === "single" && !config.startFrame) {
      message.warning(`配置${i + 1}：请选择图片`);
      return;
    }
    if (config.mode === "multi" && config.images.length === 0) {
      message.warning(`配置${i + 1}：请选择至少一张图片`);
      return;
    }
    if (!config.prompt.trim()) {
      message.warning(`配置${i + 1}：请输入视频提示词`);
      return;
    }
  }

  // 保存配置到后端，同时更新 store
  generateVideoLoading.value = true;
  try {
    for (const config of videoConfigs.value) {
      // 调用后端接口保存配置
      const res = await axios.post("/video/addVideoConfig", {
        scriptId: props.scriptId,
        projectId: Number(project.value!.id!),
        configId: config.configId,
        mode: config.mode,
        startFrame: config.startFrame,
        endFrame: config.endFrame,
        images: [...config.images],
        resolution: config.resolution,
        duration: config.duration,
        prompt: config.prompt,
      });

      // 将后端返回的配置添加到 store
      if (res.data?.data) {
        videoStoreInstance.addConfigFromBackend(res.data.data);
      }
    }
    message.success(`成功添加${videoConfigs.value.length}个视频配置`);
    storyboardShow.value = false;
  } catch (error: any) {
    message.error(error?.message || "添加配置失败");
  } finally {
    generateVideoLoading.value = false;
  }
}

function handleCancel() {
  videoConfigs.value = [];
}
</script>

<style lang="scss" scoped>
.configPanel {
  padding: 16px;
  background: #fafafa;
  border-radius: 8px;
  max-height: 70vh;
  overflow-y: auto;
  .configHeader {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 16px;
    h3 {
      margin: 0;
      font-size: 16px;
      font-weight: 600;
    }
  }
}
.configList {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16px;
}
.configCard {
  background: #fff;
  border: 1px solid #e8e8e8;
  border-radius: 8px;
  overflow: hidden;
  transition: box-shadow 0.2s;
  &:hover {
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }
  .cardHeader {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 8px 12px;
    background: #f5f5f5;
    border-bottom: 1px solid #e8e8e8;
    .cardTitle {
      font-weight: 500;
      font-size: 14px;
    }
  }
  .cardBody {
    padding: 12px;
  }
}
.formRow {
  display: flex;
  align-items: center;
  margin-bottom: 10px;
  &:last-child {
    margin-bottom: 0;
  }
  > label {
    width: 60px;
    flex-shrink: 0;
    font-size: 12px;
    color: #666;
    line-height: 24px;
  }
  > :deep(.ant-select),
  > :deep(.ant-radio-group) {
    flex: 1;
  }
  .unit {
    margin-left: 4px;
    font-size: 12px;
    color: #666;
  }
  .tip {
    margin-left: 8px;
    font-size: 11px;
    color: #999;
  }
  &.frameRow {
    align-items: flex-start;
    .frameGroup {
      display: flex;
      gap: 8px;
    }
  }
  &.promptRow {
    flex-direction: column;
    align-items: flex-start;
    > label {
      width: auto;
      margin-bottom: 4px;
    }
  }
}
.frameBox {
  width: 150px;
  height: auto;
  min-height: 70px;
  border: 1px dashed #d9d9d9;
  border-radius: 4px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s;
  position: relative;
  overflow: hidden;
  background: #fafafa;
  font-size: 10px;
  color: #999;
  &.singleFrame {
    width: 90px;
    height: 68px;
  }
  &:hover {
    border-color: #1890ff;
    background: #e6f7ff;
  }
  &.hasImage {
    border-style: solid;
    border-color: #52c41a;
    img {
      width: 100%;
      height: 100%;
      object-fit: cover;
    }
    .frameLabel {
      position: absolute;
      bottom: 0;
      left: 0;
      right: 0;
      background: rgba(0, 0, 0, 0.5);
      color: #fff;
      font-size: 10px;
      text-align: center;
      padding: 1px 0;
    }
    .removeBtn {
      position: absolute;
      top: 0;
      right: 0;
      background: rgba(255, 255, 255, 0.9);
      border-radius: 0 0 0 4px;
      width: 16px;
      height: 16px;
      padding: 0;
      display: flex;
      align-items: center;
      justify-content: center;
      opacity: 0;
      transition: opacity 0.2s;
      font-size: 10px;
    }
    &:hover .removeBtn {
      opacity: 1;
    }
  }
  span {
    margin-top: 2px;
  }
}

.multiImages {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  flex: 1;

  .imageDragList {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  .dragImageItem {
    position: relative;
    width: 150px;
    height: auto;
    border-radius: 4px;
    overflow: hidden;
    border: 1px solid #52c41a;

    .dragHandle {
      width: 100%;
      height: 100%;
      cursor: move;
      position: relative;

      .image {
        width: 100%;
        height: 100%;
        object-fit: cover;
        pointer-events: none;
        user-select: none;
      }
    }

    .imageOrder {
      position: absolute;
      top: 1px;
      left: 1px;
      background: rgba(0, 0, 0, 0.6);
      color: #fff;
      font-size: 9px;
      padding: 0 3px;
      border-radius: 2px;
      pointer-events: none;
    }

    .removeBtn {
      position: absolute;
      top: 0;
      right: 0;
      background: rgba(255, 255, 255, 0.9);
      border-radius: 0 0 0 4px;
      width: 14px;
      height: 14px;
      padding: 0;
      display: flex;
      align-items: center;
      justify-content: center;
      opacity: 0;
      transition: opacity 0.2s;
      font-size: 9px;
    }

    &:hover .removeBtn {
      opacity: 1;
    }
  }

  .ghost {
    opacity: 0.5;
    background: #c8ebfb;
  }

  .addImageBox {
    width: 150px;
    height: 70px;
    border: 1px dashed #d9d9d9;
    border-radius: 4px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    transition: all 0.2s;
    background: #fafafa;
    font-size: 12px;
    color: #999;

    &:hover {
      border-color: #1890ff;
      background: #e6f7ff;
    }
  }
}

.promptWrapper {
  position: relative;
  width: 100%;

  .magicBtn {
    position: absolute;
    right: 2px;
    top: 2px;
    font-size: 12px;
    padding: 0 4px;
    color: var(--mainColor);
  }

  :deep(.ant-input) {
    padding-right: 30px;
  }
}

.selectorFooter {
  display: flex;
  justify-content: space-between;
  align-items: center;

  .selectedCount {
    color: #1890ff;
    font-weight: 500;
  }
}

:deep(.ant-empty) {
  padding: 60px 0;
}

:deep(.ant-radio-group) {
  .ant-radio-wrapper {
    font-size: 12px;
  }
}
</style>
