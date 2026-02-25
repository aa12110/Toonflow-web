<template>
  <div class="promptEditorContainer">
    <!-- 左侧导航 -->
    <div class="promptSidebar">
      <div class="promptTypeGroup" v-for="group in groupedPrompts" :key="group.type">
        <div class="typeHeader">
          <i-tag-one theme="outline" size="14" fill="currentColor" />
          <span>{{ getTypeName(group.type) }}</span>
        </div>
        <div
          class="promptItem"
          v-for="item in group.items"
          :key="item.code"
          :class="{ active: currentPrompt?.code === item.code }"
          @click="selectPrompt(item)">
          <span class="promptName">{{ item.name }}</span>
          <span class="promptCode">{{ item.code }}</span>
        </div>
      </div>
      <div v-if="promptList.length === 0" class="emptyList">
        <i-inbox theme="outline" size="32" fill="#d1d5db" />
        <span>暂无提示词</span>
      </div>
    </div>

    <!-- 右侧编辑区 -->
    <div class="promptEditor">
      <template v-if="currentPrompt">
        <div class="editorHeader">
          <div class="editorInfo">
            <div class="titleRow">
              <h3 class="editorTitle">{{ currentPrompt.name }}</h3>
              <span :class="hasCustomValue ? 'customBadge' : 'defaultBadge'">
                {{ hasCustomValue ? "已自定义" : "默认值" }}
              </span>
            </div>
            <div class="editorMeta">
              <t-tag :color="getTypeColor(currentPrompt.type)">{{ getTypeName(currentPrompt.type) }}</t-tag>
              <span class="promptCodeBadge">{{ currentPrompt.code }}</span>
              <span v-if="currentPrompt.parentCode" class="parentInfo">父级: {{ currentPrompt.parentCode }}</span>
            </div>
          </div>
          <div class="editorActions">
            <t-button theme="default" @click="resetToDefault">
              <template #icon>
                <i-redo theme="outline" size="14" fill="currentColor" />
              </template>
              重置提示词
            </t-button>
            <t-button theme="primary" @click="savePrompt" :loading="saving">
              <template #icon>
                <i-check theme="outline" size="14" fill="currentColor" />
              </template>
              保存
            </t-button>
          </div>
        </div>

        <div class="editorBody">
          <t-textarea v-model="editingValue" placeholder="请输入提示词内容" class="promptTextarea" />
          <div class="editorTip">
            <i-info theme="outline" size="14" fill="currentColor" />
            <span>{{ hasCustomValue ? '当前使用自定义提示词，点击"重置提示词"可恢复默认值' : "当前使用默认提示词，编辑后将保存为自定义值" }}</span>
          </div>
        </div>
      </template>

      <div v-else class="emptyEditor">
        <i-edit theme="outline" size="48" fill="#d1d5db" />
        <p>请从左侧选择一个提示词进行编辑</p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from "vue";
import { message as ElMessage } from "ant-design-vue";
import axios from "@/utils/axios";

interface Prompt {
  id: number;
  code: string;
  name: string;
  type: "mainAgent" | "subAgent" | "system";
  parentCode: string;
  defaultValue: string;
  customValue: string;
}

const promptList = ref<Prompt[]>([]);
const currentPrompt = ref<Prompt | null>(null);
const editingValue = ref("");
const saving = ref(false);

const typeConfig: Record<string, { name: string; color: string }> = {
  system: { name: "系统提示词", color: "green" },
  mainAgent: { name: "主Agent", color: "purple" },
  subAgent: { name: "子Agent", color: "blue" },
};

const groupedPrompts = computed(() => {
  const groups: Record<string, Prompt[]> = {};
  promptList.value.forEach((item) => {
    (groups[item.type] ??= []).push(item);
  });
  return Object.keys(typeConfig)
    .filter((type) => groups[type])
    .map((type) => ({ type, items: groups[type] }));
});

const hasCustomValue = computed(() => {
  if (!currentPrompt.value) return false;
  const defaultVal = processLineBreaks(currentPrompt.value.defaultValue || "");
  return editingValue.value.trim() !== "" && editingValue.value !== defaultVal;
});

const getTypeName = (type: string) => typeConfig[type]?.name || type;
const getTypeColor = (type: string) => typeConfig[type]?.color || "default";
const processLineBreaks = (value: string) => value?.replace(/\\n/g, "\n") || "";

function selectPrompt(prompt: Prompt) {
  currentPrompt.value = prompt;
  editingValue.value = processLineBreaks(prompt.customValue || prompt.defaultValue || "");
}

function resetToDefault() {
  if (!currentPrompt.value) return;
  editingValue.value = processLineBreaks(currentPrompt.value.defaultValue || "");
  ElMessage.info("已重置为默认值，点击保存生效");
}

async function savePrompt() {
  if (!currentPrompt.value) return;
  saving.value = true;
  try {
    const defaultVal = processLineBreaks(currentPrompt.value.defaultValue || "");
    const customVal = editingValue.value === defaultVal ? "" : editingValue.value.trim();

    await axios.post("/prompt/updatePrompt", {
      id: currentPrompt.value.id,
      code: currentPrompt.value.code,
      customValue: customVal,
    });

    const index = promptList.value.findIndex((p) => p.code === currentPrompt.value?.code);
    if (index !== -1) promptList.value[index].customValue = customVal;
    ElMessage.success("保存成功");
  } catch {
    ElMessage.error("保存失败");
  } finally {
    saving.value = false;
  }
}

async function fetchPrompts() {
  try {
    const res = await axios.get("/prompt/getPrompts");
    promptList.value = res.data || [];
    if (promptList.value.length > 0) selectPrompt(promptList.value[0]);
  } catch {
    ElMessage.error("获取提示词列表失败");
  }
}

onMounted(fetchPrompts);
</script>

<style lang="scss" scoped>
.promptEditorContainer {
  display: flex;
  height: 65vh;
}

.promptSidebar {
  width: 240px;
  border-right: 1px solid var(--td-border-level-1-color);
  display: flex;
  flex-direction: column;
  background: var(--td-bg-color-secondarycontainer);
  overflow: auto;
}

.promptTypeGroup {
  padding: 8px 0;
}

.typeHeader {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 16px;
  font-size: 12px;
  font-weight: 500;
  color: var(--td-text-color-secondary);
  text-transform: uppercase;
}

.promptItem {
  display: flex;
  flex-direction: column;
  padding: 10px 16px;
  cursor: pointer;
  transition: all 0.2s;
  border-left: 3px solid transparent;

  &:hover,
  &.active {
    background: var(--td-brand-color-light);
  }

  &.active {
    border-left-color: var(--td-brand-color);
  }
}

.promptName {
  font-size: 13px;
  color: var(--td-text-color-primary);
  font-weight: 500;
}

.promptCode {
  font-size: 11px;
  color: var(--td-text-color-placeholder);
  font-family: monospace;
}

.emptyList {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  flex: 1;
  color: var(--td-text-color-placeholder);
  gap: 8px;
}

.promptEditor {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.editorHeader {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  padding: 16px 20px;
  border-bottom: 1px solid var(--td-border-level-1-color);
  gap: 16px;
}

.editorInfo {
  flex: 1;
  min-width: 0;
}

.titleRow {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 8px;
}

.editorTitle {
  font-size: 16px;
  font-weight: 600;
  color: var(--td-text-color-primary);
  margin: 0;
}

.customBadge,
.defaultBadge {
  font-size: 11px;
  padding: 3px 10px;
  border-radius: 12px;
  font-weight: 500;
}

.customBadge {
  color: var(--td-warning-color);
  background: var(--td-warning-color-light);
}

.defaultBadge {
  color: var(--td-success-color);
  background: var(--td-success-color-light);
}

.editorMeta {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.promptCodeBadge {
  font-size: 12px;
  color: var(--td-text-color-secondary);
  background: var(--td-bg-color-secondarycontainer);
  padding: 2px 8px;
  border-radius: 4px;
  font-family: monospace;
}

.parentInfo {
  font-size: 12px;
  color: var(--td-text-color-placeholder);
}

.editorActions {
  display: flex;
  gap: 8px;
  flex-shrink: 0;

  .ant-btn {
    display: inline-flex;
    align-items: center;
    gap: 4px;
  }
}

.editorBody {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  display: flex;
  flex-direction: column;
}

.promptTextarea {
  font-size: 13px;
  line-height: 1.7;
  border-radius: 8px;
  font-family: "SF Mono", Monaco, Menlo, Consolas, monospace;

  &:focus {
    border-color: var(--td-brand-color);
    box-shadow: 0 0 0 3px var(--td-brand-color-focus);
  }
}

.editorTip {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: var(--td-text-color-placeholder);
  padding-top: 12px;
}

.emptyEditor {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: var(--td-text-color-placeholder);

  p {
    margin: 16px 0 0 0;
    font-size: 14px;
  }
}

:deep(.ant-btn-primary) {
  background: var(--mainGradient);
  border: none;

  &:hover {
    background: var(--mainGradientHover);
  }
}
:deep(.t-textarea__inner) {
  height: 48vh;
}
</style>
