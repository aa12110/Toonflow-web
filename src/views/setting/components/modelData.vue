<template>
  <div class="modelData">
    <a-modal v-model:open="modelDataShow" title="模型数据" :footer="null" width="90%">
      <div class="data">
        <a-button style="margin-bottom: 10px" type="primary" ghost @click="addModelBtn">新增模型</a-button>
        <vxe-table ref="tableRef" :data="tableData" :radio-config="{ highlight: true }">
          <vxe-column type="radio" title="选中" width="60"></vxe-column>
          <vxe-column field="manufacturer" title="厂商" width="100"></vxe-column>
          <vxe-column field="modelType" title="类型" width="150">
            <template #default="{ row }">
              <span v-if="row.type == 'text'">文本模型</span>
              <span v-if="row.type == 'image'">图像模型</span>
              <span v-if="row.type == 'video'">视频模型</span>
            </template>
          </vxe-column>
          <vxe-column field="model" title="模型" width="250"></vxe-column>
          <vxe-column field="baseUrl" title="Base URL"></vxe-column>
          <vxe-column field="apiKey" title="API Key">
            <template #default="{ row }">
              <div class="f">
                <div style="margin-right: 8px; font-size: 15px">
                  {{ visibleMap[row.id] ? row.apiKey : "********" }}
                </div>
                <i-preview-open
                  v-if="!visibleMap[row.id]"
                  theme="outline"
                  size="20"
                  fill="#9b9b9b"
                  style="cursor: pointer"
                  @click="setVisible(row.id, true)" />
                <i-preview-close v-else theme="outline" size="20" fill="#9b9b9b" style="cursor: pointer" @click="setVisible(row.id, false)" />
              </div>
            </template>
          </vxe-column>
          <vxe-column field="createTime" title="创建时间" width="160">
            <template #default="{ row }">
              <div>{{ dayjs(row.createTime).format("YYYY-MM-DD HH:mm:ss") }}</div>
            </template>
          </vxe-column>
          <vxe-column :min-width="40" title="操作" align="center" width="200" show-overflow>
            <template #default="{ row }">
              <el-button type="warning" size="small" :loading="row.load" @click="testAi(row)">测试</el-button>
              <el-button type="info" size="small" @click="editModelBtn(row)">编辑</el-button>
              <el-button type="danger" size="small" style="margin-left: 10px" @click="delModelBtn(row)">删除</el-button>
            </template>
          </vxe-column>
        </vxe-table>
      </div>
      <div class="footer">
        <a-button style="margin-bottom: 10px; margin-left: 10px" type="primary" ghost @click="confirmConfig">确认配置</a-button>
      </div>
    </a-modal>
    <newAddModel v-model:modelShow="modelShow" v-model:modelForm="modelForm" :state="state" @fetchModelList="fetchModelList" />
    <!-- 图像测试结果预览弹窗 -->
    <a-modal title="图像生成测试成功" v-model:open="testImageModalVisible" :footer="null" centered width="auto" class="test-image-modal">
      <div class="test-image-content">
        <p class="test-image-tip">✅ 图像模型配置正确，以下是生成的测试图片：</p>
        <a-image :src="testImageResult" :preview="{ src: testImageResult }" class="test-image-preview" />
      </div>
    </a-modal>
    <a-modal title="视频生成测试成功" v-model:open="testVideoVisible" :footer="null" centered width="auto" class="test-image-modal">
      <div class="test-image-content">
        <p class="test-image-tip">✅ 视频模型配置正确，以下是生成的测试视频：</p>
        <video :src="testVideoResult" controls class="test-image-preview" />
      </div>
    </a-modal>
  </div>
</template>

<script setup lang="ts">
import { ref } from "vue";
import dayjs from "dayjs";
import { message, Modal } from "ant-design-vue";
import axios from "@/utils/axios";
import newAddModel from "./addModel.vue";
const modelDataShow = defineModel("modelDataShow", {
  type: Boolean,
  required: true,
});
interface ModelType {
  id: number;
  model: string;
  name: string;
}
const configingModel = defineModel<ModelType>("configingModel");
interface RowData {
  id: number;
  name: string;
  type: string;
  modelType: string;
  model: string;
  baseUrl: string;
  manufacturer: string;
  createTime: number;
  apiKey: string;
  load?: boolean;
}
const tableRef = ref();
const state = ref("");
const modelShow = ref(false);
const tableData = ref<RowData[]>([]);
//模型表单数据
const modelForm = ref<RowData>({
  id: 0,
  name: "",
  type: "",
  modelType: "",
  model: "",
  apiKey: "",
  baseUrl: "",
  manufacturer: "",
  createTime: 0,
  load: false,
});
//新增模型
function addModelBtn() {
  state.value = "新增模型";
  modelForm.value = {
    id: 0,
    name: "",
    type: "",
    modelType: "",
    model: "",
    apiKey: "",
    baseUrl: "",
    manufacturer: "",
    createTime: 0,
  };
  modelShow.value = true;
}
const visibleMap = reactive<Record<string | number, boolean>>({});
function setVisible(id: string | number, val: boolean) {
  visibleMap[id] = val;
}
const testImageResult = ref<string>("");
const testImageModalVisible = ref(false);

const testVideoVisible = ref(false);
const testVideoResult = ref<string>("");
async function testAi(row: RowData) {
  console.log("%c Line:124 🍎 row", "background:#6ec1c2", row);
  const { model, apiKey, baseUrl, manufacturer } = row;

  if (!model) {
    message.warning("请先填写模型名称");
    return;
  }
  if (!apiKey) {
    message.warning("请先填写 API Key");
    return;
  }
  if (!manufacturer) {
    message.warning("请先选择厂商");
    return;
  }
  row.load = true;

  try {
    let queryUrl = "";
    if (row.type == "text") {
      queryUrl = "/other/testAI";
    } else if (row.type == "image") {
      queryUrl = "/other/testImage";
    } else if (row.type == "video") {
      queryUrl = "/other/testVideo";
    }
    if (!queryUrl) {
      row.load = false;
      return message.warning("type 错误");
    }
    const res = await axios.post(queryUrl, {
      modelName: model,
      apiKey: apiKey,
      baseURL: baseUrl || undefined,
      manufacturer,
    });
    console.log("%c Line:164 🍺 res", "background:#93c0a4", res);

    if (row.type == "text") {
      message.success("连接成功！模型配置正确");
    }
    if (row.type == "image") {
      message.success("连接成功！图像模型配置正确");
      // 显示生成的图片
      if (res.data) {
        testImageResult.value = res.data;
        testImageModalVisible.value = true;
      }
    }
    if (row.type == "video") {
      message.success("连接成功！视频模型配置正确");
      if (res.data) {
        testVideoResult.value = res.data;
        testVideoVisible.value = true;
      }
    }
  } catch (e: any) {
    message.error(`连接失败: ${e.message}`);
  } finally {
    row.load = false;
  }
}
//编辑模型
function editModelBtn(row: RowData) {
  state.value = "编辑模型";
  modelForm.value = { ...row };
  modelShow.value = true;
}
watch(
  () => modelDataShow.value,
  (val) => {
    if (val == true) {
      fetchModelList();
    }
  },
  { deep: true },
);
//查询模型列表
async function fetchModelList() {
  const res = await axios.post("/setting/getSetting");
  tableData.value = res.data;
}

//删除模型
function delModelBtn(row: RowData) {
  axios
    .post("/setting/delModel", { id: row.id })
    .then(() => {
      message.success("项目删除成功");
      fetchModelList();
      emit("modelList");
    })
    .catch(() => {
      message.error("项目删除失败");
    });
}
const emit = defineEmits(["modelList"]);
// 确认配置
async function confirmConfig() {
  const selectedRow = tableRef.value?.getRadioRecord();
  if (!selectedRow) {
    message.warning("请先选择一个模型");
    return;
  }
  try {
    await axios.post("/setting/configurationModel", {
      id: configingModel.value?.id,
      configId: selectedRow.id,
    });

    message.success("配置成功");
    tableRef.value?.clearRadioRow();
    modelDataShow.value = false;
    emit("modelList");
  } catch {
    message.error("配置失败，请重试");
  }
}
</script>

<style lang="scss" scoped>
.footer {
  text-align: right;
  margin-top: 16px;
}
</style>
