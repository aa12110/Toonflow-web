<template>
  <a-modal v-model:open="modalVisible" title="视频详情" width="90vw" :style="{ top: '20px' }" :footer="null" destroyOnClose>
    <div class="video-detail" v-if="config">
      <!-- 左侧：配置信息和生成按钮 -->
      <div class="left-panel">
        <div class="config-section">
          <h3 class="section-title">配置信息</h3>
          <!-- 使用公共组件进行配置编辑 -->
          <VideoConfigForm
            v-if="editableConfig"
            :config="editableConfig"
            :script-id="config.scriptId"
            :editable="true"
            :manufacturer-disabled="true"
            @change="handleConfigFormChange" />
        </div>

        <!-- 生成按钮 -->
        <div class="action-section">
          <a-button type="primary" size="large" block :loading="isGenerating" @click="handleGenerate">
            <template #icon><i-video-two /></template>
            生成视频
          </a-button>
          <p class="action-hint">每次生成可能产生不同的结果，可多次尝试</p>
        </div>
      </div>

      <!-- 右侧：生成结果列表 -->
      <div class="right-panel">
        <h3 class="section-title">
          生成结果
          <span class="result-count" v-if="results.length">{{ results.length }}个</span>
        </h3>

        <div class="results-list" v-if="results.length">
          <div
            v-for="result in results"
            :key="result.id"
            class="result-card"
            :class="{
              selected: config.selectedResultId === result.id,
              success: result.state === 1,
              generating: result.state === 0,
              failed: result.state === -1,
            }"
            @click="handleSelectResult(result)">
            <!-- 成功状态 -->
            <template v-if="result.state === 1">
              <div class="video-cover" @click.stop="playVideo(result)">
                <img v-if="result.firstFrame" :src="result.firstFrame" alt="视频封面" />
                <video v-else-if="result.filePath" :src="result.filePath" preload="metadata"></video>
                <div v-else class="video-placeholder">
                  <i-film :size="32" />
                  <span>视频</span>
                </div>
                <div class="play-overlay">
                  <i-play-one theme="filled" :size="24" fill="#fff" />
                </div>
                <div class="duration-badge" v-if="result.duration">
                  {{ formatDuration(result.duration) }}
                </div>
              </div>
              <div class="result-actions">
                <a-button v-if="config.selectedResultId !== result.id" type="primary" size="small" @click.stop="handleSelectResult(result)">
                  选择此视频
                </a-button>
                <span v-else class="selected-badge">
                  <i-check-one theme="filled" :size="14" />
                  已选择
                </span>
              </div>
            </template>

            <!-- 生成中状态 -->
            <template v-else-if="result.state === 0">
              <div class="status-cover generating">
                <div class="loading-spinner"></div>
                <span>生成中...</span>
              </div>
            </template>

            <!-- 失败状态 -->
            <template v-else>
              <div class="status-cover failed">
                <i-close-one theme="filled" :size="24" fill="#ef4444" />
                <span>生成失败</span>
              </div>
            </template>
          </div>
        </div>

        <!-- 空状态 -->
        <div class="empty-results" v-else>
          <div class="empty-icon">
            <i-film :size="48" />
          </div>
          <p class="empty-title">暂无生成结果</p>
          <p class="empty-desc">点击左侧按钮开始生成视频</p>
        </div>
      </div>
    </div>

    <!-- 视频播放弹窗 -->
    <a-modal v-model:open="videoPlayerVisible" :title="null" :footer="null" width="800px" centered destroyOnClose wrapClassName="video-player-modal">
      <div class="video-player-content">
        <video v-if="currentPlayVideo" ref="videoRef" :src="currentPlayVideo.filePath" controls autoplay class="video-element" />
      </div>
    </a-modal>
  </a-modal>
</template>

<script setup lang="ts">
import { ref, computed, watch } from "vue";
import { message } from "ant-design-vue";
import videoStore, { type VideoResult } from "@/stores/video";
import { storeToRefs } from "pinia";
import axios from "@/utils/axios";
import { VideoConfigForm, type VideoConfigData } from "@/components/videoConfig";

const props = defineProps<{
  configId: number | null;
}>();

const modalVisible = defineModel<boolean>({});
const store = videoStore();
const { videoConfigs, videoResults } = storeToRefs(store);

const isGenerating = ref(false);
const videoPlayerVisible = ref(false);
const currentPlayVideo = ref<VideoResult | null>(null);
const videoRef = ref<HTMLVideoElement | null>(null);

// 可编辑配置（将 VideoConfig 转换为 VideoConfigData 格式）
const editableConfig = ref<VideoConfigData | null>(null);

// 当前配置
const config = computed(() => {
  if (!props.configId) return null;
  console.log("%c Line:139 🍐 props.configId", "background:#2eafb0", props.configId);
  console.log(
    "%c Line:142 🍩 videoConfigs.value.find((c) => c.id === props.configId)",
    "background:#465975",
    videoConfigs.value.find((c) => c.id === props.configId),
  );

  return videoConfigs.value.find((c) => c.id === props.configId) || null;
});

// 当前配置的所有结果
const results = computed(() => {
  if (!props.configId) return [];
  return store.getResultsByConfigId(props.configId);
});

// 监听配置变化，初始化可编辑配置
watch(
  config,
  (newConfig) => {
    if (newConfig) {
      console.log("%c Line:160 🍺 newConfig", "background:#ffdd4d", newConfig);
      editableConfig.value = {
        id: newConfig.id,
        manufacturer: newConfig.manufacturer,
        model: newConfig.model,
        aiConfigId: newConfig.aiConfigId,
        mode: newConfig.mode,
        startFrame: newConfig.startFrame,
        endFrame: newConfig.endFrame,
        images: newConfig.images ? [...newConfig.images] : [],
        resolution: newConfig.resolution,
        duration: newConfig.duration,
        prompt: newConfig.prompt,
        audioEnabled: newConfig.audioEnabled,
      };
    } else {
      editableConfig.value = null;
    }
  },
  { immediate: true },
);

// 配置表单变更处理
async function handleConfigFormChange(updatedConfig: VideoConfigData) {
  if (!props.configId || !config.value) return;

  // 更新可编辑配置
  editableConfig.value = updatedConfig;

  // 更新本地 store（包括图片变更）
  store.updateConfigFull(props.configId, {
    resolution: updatedConfig.resolution,
    duration: updatedConfig.duration,
    prompt: updatedConfig.prompt,
    startFrame: updatedConfig.startFrame,
    endFrame: updatedConfig.endFrame,
    images: updatedConfig.images,
    mode: updatedConfig.mode,
    audioEnabled: updatedConfig.audioEnabled,
  });

  // 调用后端接口更新配置
  try {
    await axios.post("/video/upDateVideoConfig", {
      id: props.configId,
      resolution: updatedConfig.resolution,
      duration: updatedConfig.duration,
      prompt: updatedConfig.prompt,
      startFrame: updatedConfig.startFrame,
      endFrame: updatedConfig.endFrame,
      images: updatedConfig.images,
      audioEnabled: updatedConfig.audioEnabled,
    });
  } catch (error: any) {
    console.error("更新配置失败:", error);
  }
}

function formatDuration(seconds: number): string {
  const mins = Math.floor(seconds / 60);
  const secs = Math.floor(seconds % 60);
  return `${mins}:${secs.toString().padStart(2, "0")}`;
}

// 生成视频
async function handleGenerate() {
  if (!props.configId) return;

  isGenerating.value = true;
  try {
    await store.generateVideo(props.configId);
    message.success("视频生成任务已提交");
  } catch (error: any) {
    message.error(error?.message || "生成失败");
  } finally {
    isGenerating.value = false;
  }
}

// 选择结果
async function handleSelectResult(result: VideoResult) {
  if (result.state !== 1 || !props.configId) return;

  // 更新本地 store
  store.selectResult(props.configId, result.id);

  // 调用后端接口更新选中结果
  try {
    await axios.post("/video/upDateVideoConfig", {
      id: props.configId,
      selectedResultId: result.id,
    });
    message.success("已选择此视频");
  } catch (error: any) {
    message.error("选择失败");
    console.error("更新选中结果失败:", error);
  }
}

// 播放视频
function playVideo(result: VideoResult) {
  if (result.state !== 1 || !result.filePath) return;
  currentPlayVideo.value = result;
  videoPlayerVisible.value = true;
}

// 关闭播放器时暂停视频
watch(videoPlayerVisible, (visible) => {
  if (!visible && videoRef.value) {
    videoRef.value.pause();
    currentPlayVideo.value = null;
  }
});
</script>

<style lang="scss" scoped>
.video-detail {
  display: flex;
  gap: 24px;
  height: 88vh;
  overflow: auto;

  .left-panel {
    width: 450px;
    flex-shrink: 0;
    display: flex;
    flex-direction: column;
    gap: 20px;

    .config-section {
      background: #f9fafb;
      border-radius: 12px;
      padding: 20px;

      .section-title {
        margin: 0 0 16px;
        font-size: 16px;
        font-weight: 600;
        color: #1f2937;
      }
    }

    .action-section {
      .action-hint {
        margin: 10px 0 0;
        font-size: 12px;
        color: #9ca3af;
        text-align: center;
      }
    }
  }

  .right-panel {
    flex: 1;
    min-width: 0;

    .section-title {
      display: flex;
      align-items: center;
      gap: 10px;
      margin: 0 0 16px;
      font-size: 16px;
      font-weight: 600;
      color: #1f2937;

      .result-count {
        padding: 2px 10px;
        background: rgba(147, 51, 234, 0.1);
        color: #9333ea;
        border-radius: 20px;
        font-size: 12px;
        font-weight: 500;
      }
    }

    .results-list {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
      gap: 16px;

      .result-card {
        background: #fff;
        border-radius: 12px;
        overflow: hidden;
        border: 2px solid transparent;
        transition: all 0.2s ease;
        cursor: pointer;

        &:hover {
          border-color: rgba(147, 51, 234, 0.3);
          box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
        }

        .video-cover {
          position: relative;
          width: 100%;
          height: 140px;
          overflow: hidden;

          img,
          video {
            width: 100%;
            height: 100%;
            object-fit: cover;
          }

          .video-placeholder {
            width: 100%;
            height: 100%;
            background: linear-gradient(135deg, #f3e8ff, #e9d5ff);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            gap: 8px;
            color: #9333ea;

            span {
              font-size: 13px;
              font-weight: 500;
            }
          }

          .play-overlay {
            position: absolute;
            inset: 0;
            background: rgba(0, 0, 0, 0.3);
            display: flex;
            align-items: center;
            justify-content: center;
            opacity: 0;
            transition: opacity 0.2s ease;
          }

          &:hover .play-overlay {
            opacity: 1;
          }

          .duration-badge {
            position: absolute;
            bottom: 8px;
            right: 8px;
            padding: 2px 8px;
            background: rgba(0, 0, 0, 0.75);
            color: #fff;
            border-radius: 4px;
            font-size: 11px;
          }
        }

        .result-actions {
          padding: 12px;
          display: flex;
          justify-content: center;

          .selected-badge {
            display: flex;
            align-items: center;
            gap: 4px;
            color: #9333ea;
            font-size: 13px;
            font-weight: 500;
          }
        }

        .status-cover {
          height: 140px;
          display: flex;
          flex-direction: column;
          align-items: center;
          justify-content: center;
          gap: 10px;

          &.generating {
            background: linear-gradient(135deg, #f3e8ff, #e9d5ff);

            .loading-spinner {
              width: 32px;
              height: 32px;
              border: 3px solid #e5e7eb;
              border-top-color: #9333ea;
              border-radius: 50%;
              animation: spin 1s linear infinite;
            }

            span {
              color: #7c3aed;
              font-size: 13px;
            }
          }

          &.failed {
            background: #fef2f2;

            span {
              color: #ef4444;
              font-size: 13px;
            }
          }
        }
      }
    }

    .empty-results {
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      padding: 60px 20px;
      background: #f9fafb;
      border-radius: 12px;
      border: 2px dashed #e5e7eb;

      .empty-icon {
        width: 80px;
        height: 80px;
        background: linear-gradient(135deg, #f3e8ff, #e9d5ff);
        border-radius: 50%;
        display: flex;
        align-items: center;
        justify-content: center;
        color: #9333ea;
        margin-bottom: 16px;
      }

      .empty-title {
        margin: 0 0 8px;
        font-size: 16px;
        font-weight: 600;
        color: #374151;
      }

      .empty-desc {
        margin: 0;
        font-size: 13px;
        color: #9ca3af;
      }
    }
  }
}

.video-player-content {
  .video-element {
    width: 100%;
    max-height: 70vh;
    display: block;
    background: #000;
    border-radius: 8px;
  }
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
</style>
