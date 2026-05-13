<template>
  <div class="aiConfog" v-loading="loading">
    <div class="banner">
      <div class="content f ac jb">
        <div class="textContent ac">
          <i-good-two class="icon" theme="filled" size="24" fill="currentColor" />
          <span>{{ $t("settings.agent.bannerDesc") }}</span>
        </div>
        <div class="btnList f w">
          <t-button @click="jumpToWebsite">
            {{ $t("settings.agent.visitWebsite") }}
            <template #suffix>
              <i-share theme="outline" />
            </template>
          </t-button>
          <div class="rightBtnList f nw">
            <t-button @click="oneClickToFillIn">{{ $t("settings.agent.oneClickFill") }}</t-button>
            <t-button theme="primary" variant="outline" @click="openBatchConfig">批量切换模型</t-button>
          </div>
        </div>
      </div>
    </div>

    <t-tabs :default-value="1">
      <t-tab-panel :value="1" :label="$t('settings.agent.ordinary')">
        <div class="cardGrid">
          <t-card hoverShadow v-for="(item, index) in modelData" :key="index" class="skillCard f" @click="startConfig(item, '普通')">
            <div class="skillCardHeader">
              <div class="headerLeft">
                <t-avatar v-if="getDisplayLogo(item)" :image="getDisplayLogo(item)!" shape="round" />
                <t-avatar v-else shape="round" class="fallbackAvatar">
                  {{ getFallbackText(item.name) }}
                </t-avatar>
                <span class="skillName">{{ item.name }}</span>
              </div>
              <t-tag v-if="item.model && !item.disabled" theme="primary" variant="light" size="small">{{ item.model }}</t-tag>
              <t-tag v-else-if="item.disabled" variant="light" size="small">{{ $t("settings.agent.notOpen") }}</t-tag>
              <t-tag v-else-if="!item.disabled && !item.model" theme="warning" variant="light" size="small">
                {{ $t("settings.agent.notConfigured") }}
              </t-tag>
            </div>
            <div class="skillCardBody">{{ item.desc }}</div>
          </t-card>
        </div>
      </t-tab-panel>
      <t-tab-panel :value="2" :label="$t('settings.agent.advanced')">
        <div style="display: flex; justify-content: flex-end; margin-top: 8px; margin-bottom: -8px;">
          <t-popconfirm content="清空所有高级配置？清空后子 Agent 将继承简易配置中父级的模型设置。" @confirm="clearAllAdvanced">
            <t-button theme="danger" variant="text" size="small" :loading="clearingAdvanced">
              {{ clearingAdvanced ? '清空中…' : '清空所有配置' }}
            </t-button>
          </t-popconfirm>
        </div>
        <div class="cardGrid">
          <t-card hoverShadow v-for="(item, index) in advancedModelData" :key="index" class="skillCard f" @click="startConfig(item, '高级')">
            <div class="skillCardHeader">
              <div class="headerLeft">
                <t-avatar v-if="getDisplayLogo(item)" :image="getDisplayLogo(item)!" shape="round" />
                <t-avatar v-else shape="round" class="fallbackAvatar">
                  {{ getFallbackText(item.name) }}
                </t-avatar>
                <div>
                  <div class="skillName">{{ item.name }}</div>
                </div>
              </div>
              <t-tag v-if="item.model && !item.disabled" theme="primary" variant="light" size="small">{{ item.model }}</t-tag>
              <t-tag v-else-if="item.disabled" variant="light" size="small">{{ $t("settings.agent.notOpen") }}</t-tag>
              <t-tag v-else-if="!item.disabled && !item.model" theme="warning" variant="light" size="small">
                {{ $t("settings.agent.notConfigured") }}
              </t-tag>
            </div>
            <div class="skillCardBody jb">
              <div>{{ item.desc }}</div>
              <div>
                <t-tag theme="primary" variant="light" size="small" style="margin-left: 5px;">{{$t('settings.agent.temperature')}}：{{ item.temperature }}</t-tag>
                <t-tag :theme="item.maxOutputTokens === 0 ? 'success' : 'primary'" variant="light" size="small" style="margin-left: 5px;">
                  {{ $t('settings.agent.maxOutputTokens') }}：{{ item.maxOutputTokens === 0 ? $t('settings.agent.auto') : item.maxOutputTokens }}
                </t-tag>
              </div>
            </div>
          </t-card>
        </div>
      </t-tab-panel>
    </t-tabs>

    <!-- 批量切换模型弹窗 -->
    <t-dialog
      v-model:visible="batchModelDataShow"
      header="批量切换所有 Agent 的模型"
      width="540px"
      :on-confirm="confirmBatchConfig"
      :on-close="() => (batchModelDataShow = false)"
      :confirm-loading="batchApplying"
      :confirm-btn="batchApplying ? '应用中…' : `应用到 ${batchTargetCount} 个 Agent`"
      :cancel-btn="$t('settings.agent.cancel')">
      <div class="dialogContent">
        <t-form label-align="top" :label-width="70">
          <t-form-item label="供应商 / 模型">
            <modelSelect v-model="batchSelectValue" v-model:label="batchSelectLabel" type="text" />
          </t-form-item>
          <t-form-item label="影响范围">
            <div style="font-size: 13px; color: var(--td-text-color-secondary); line-height: 1.6">
              将把 <strong>{{ batchTargetCount }}</strong> 个 Agent（普通 {{ modelData.filter((m) => !m.disabled).length }} + 高级
              {{ advancedModelData.filter((m) => !m.disabled).length }}）全部改成此模型。<br />
              已禁用的 Agent（如 TTS 配音、导演规划）保持不变。
            </div>
          </t-form-item>
          <t-form-item label="测试连通">
            <div style="display: flex; align-items: center; gap: 8px; flex-wrap: wrap">
              <t-button :disabled="!batchSelectValue || batchTesting" :loading="batchTesting" @click="testBatchModel">
                测试一下（不会改动配置）
              </t-button>
              <t-tag v-if="batchTestStatus === 'success'" theme="success" variant="light" size="small">连通 ✓</t-tag>
              <t-tag v-else-if="batchTestStatus === 'error'" theme="danger" variant="light" size="small">失败</t-tag>
            </div>
            <div
              v-if="batchTestMsg"
              :style="{
                marginTop: '8px',
                padding: '8px 10px',
                borderRadius: '4px',
                background: 'var(--td-bg-color-container-hover)',
                fontSize: '12px',
                lineHeight: '1.5',
                whiteSpace: 'pre-wrap',
                wordBreak: 'break-word',
                maxHeight: '120px',
                overflow: 'auto',
              }">
              {{ batchTestMsg }}
            </div>
          </t-form-item>
        </t-form>
      </div>
    </t-dialog>

    <!-- 模型配置弹窗 -->
    <t-dialog
      v-model:visible="modelDataShow"
      :header="currentItem?.name + ' ' + $t('settings.agent.modelConfig')"
      width="480px"
      :on-confirm="confirmConfig"
      :confirm-btn="$t('settings.agent.confirm')"
      :cancel-btn="$t('settings.agent.cancel')">
      <div class="dialogContent">
        <t-form v-if="currentItem" label-align="top" :label-width="70">
          <t-form-item :label="$t('settings.agent.selectModel')">
            <modelSelect v-model="selectValue" v-model:label="selectLabel" type="text" />
          </t-form-item>
          <t-form-item :label="$t('settings.agent.temperature')" v-if="type == '高级'">
            <t-input-number v-model="currentItem.temperature" style="width: 100%" />
          </t-form-item>
          <t-form-item :label="$t('settings.agent.maxOutputTokens')" v-if="type == '高级'">
            <div class="maxTokenRow">
              <t-radio-group v-model="maxTokenMode" variant="default-filled" size="small">
                <t-radio-button value="auto">{{ $t('settings.agent.auto') }}</t-radio-button>
                <t-radio-button value="manual">{{ $t('settings.agent.manual') }}</t-radio-button>
              </t-radio-group>
              <t-input-number
                v-if="maxTokenMode === 'manual'"
                v-model="currentItem.maxOutputTokens"
                :min="1"
                theme="normal"
                style="flex: 1; margin-left: 12px;"
              />
              <span v-else class="autoHint">{{ $t('settings.agent.autoHint') }}</span>
            </div>
          </t-form-item>
        </t-form>
      </div>
    </t-dialog>
  </div>
</template>

<script setup lang="ts">
import modelSelect from "@/components/modelSelect.vue";
import { providersLogo, modelProviderRules } from "@/utils/providersLogo";
import axios from "@/utils/axios";
import settingStore from "@/stores/setting";
const { isElectron } = storeToRefs(settingStore());

interface ModelType {
  id: number;
  model: string;
  modelName: string;
  vendorId: number | null;
  name: string;
  icon: string;
  desc: string;
  disabled?: boolean;
  temperature?: number;
  maxOutputTokens?: number;
}

const modelData = ref<ModelType[]>([]);

const modelDataShow = ref(false);
const currentItem = ref<ModelType | null>(null);
const selectValue = ref<string>("");
const selectLabel = ref<string>("");

function getProviderLogo(manufacturer: string) {
  if (!manufacturer) return null;
  const key = Object.keys(providersLogo).find((k) => k.toLowerCase() === manufacturer.toLowerCase());
  return key ? providersLogo[key as keyof typeof providersLogo] : null;
}

function inferProviderByModel(modelName?: string, model?: string) {
  const source = `${modelName || ""} ${model || ""}`.trim();
  if (!source) return null;
  const matchedRule = modelProviderRules.find((rule: { pattern: RegExp }) => rule.pattern.test(source));
  return matchedRule ? providersLogo[matchedRule.provider] : null;
}

function getDisplayLogo(item: ModelType) {
  return getProviderLogo(item.icon) || inferProviderByModel(item.modelName, item.model);
}

function getFallbackText(name: string) {
  return name?.slice(0, 1) || "A";
}
const type = ref("");
const maxTokenMode = ref<"auto" | "manual">("auto");

watch(maxTokenMode, (val) => {
  if (val === "auto" && currentItem.value) {
    currentItem.value.maxOutputTokens = 0;
  }
  if (val === "manual" && currentItem.value && (currentItem.value.maxOutputTokens === 0 || currentItem.value.maxOutputTokens == null)) {
    currentItem.value.maxOutputTokens = 8192;
  }
});

function startConfig(item: ModelType, source: string) {
  if (item.disabled) return window.$message.warning($t("settings.agent.msg.notAvailable"));
  currentItem.value = item;
  selectValue.value = item.modelName || "";
  selectLabel.value = item.model || "";
  maxTokenMode.value = (item.maxOutputTokens === 0 || item.maxOutputTokens == null) ? "auto" : "manual";
  modelDataShow.value = true;
  type.value = source;
}

const currentVendorId = ref<number | null>(null);
function confirmConfig() {
  if (currentItem.value) {
    currentItem.value.model = selectLabel.value;
    currentItem.value.modelName = selectValue.value;
    currentItem.value.vendorId = currentVendorId.value;
  }
  const data = {
    id: currentItem.value?.id,
    name: currentItem.value?.name,
    model: selectLabel.value || selectValue.value.split(/:(.+)/)[1] || currentItem.value?.model,
    modelName: currentItem.value?.modelName,
    vendorId: selectValue.value.split(/:(.+)/)[0],
    desc: currentItem.value?.desc,
    temperature: currentItem.value?.temperature ?? 1,
    maxOutputTokens: currentItem.value?.maxOutputTokens ?? 0,
  };
  axios
    .post("/setting/agentDeploy/deployAgentModel", data)
    .then(() => {
      window.$message.success($t("settings.agent.msg.configSuccess"));
      getAgentDeploy();
    })
    .catch((err: { message?: string }) => {
      window.$message.error(`${$t("settings.agent.msg.updateConfigFailed")}${err.message}`);
    })
    .finally(() => {
      modelDataShow.value = false;
    });
}
//跳转官方网站
async function jumpToWebsite() {
  if (isElectron.value) {
    await fetch(`toonflow://openurlwithbrowser?url=https://api.toonflow.net`);
  } else {
    window.open("https://api.toonflow.net", "_blank");
  }
}
const loading = ref(false);

//查询Agent配置列表
function getAgentDeploy() {
  axios
    .post("/setting/agentDeploy/getAgentDeploy")
    .then((res: any) => {
      modelData.value = res.data.qrdinaryData;
      advancedModelData.value = res.data.advancedData;
    })
    .catch((err: { message?: string }) => {
      window.$message.error(`${$t("settings.agent.msg.getAgentListFailed")}${err.message}`);
    })
    .finally(() => {});
}
onMounted(() => {
  getAgentDeploy();
});
//一键填入
async function oneClickToFillIn() {
  loading.value = true;
  await getVendorList();
  const toonflow = vendorList.value.find((item) => item.id === "toonflow");
  if (!toonflow) {
    window.$message.error($t("settings.agent.msg.toonflowNotFound"));
    loading.value = false;
    return;
  }
  if (!toonflow.inputValues.apiKey) {
    // key 不存在，弹窗让用户填入
    loading.value = false;
    const inputKey = ref("");
    const dialogInstance = DialogPlugin({
      theme: "warning",
      header: $t("settings.agent.fillKeyHeader"),
      body: () =>
        h("div", { style: "padding: 8px 0" }, [
          h(resolveComponent("t-input") as any, {
            modelValue: inputKey.value,
            "onUpdate:modelValue": (val: string) => (inputKey.value = val),
            placeholder: $t("settings.agent.keyPlaceholder"),
            type: "password",
          }),
        ]),
      confirmBtn: $t("settings.agent.confirm"),
      cancelBtn: $t("settings.agent.cancel"),
      onConfirm: () => {
        if (!inputKey.value) {
          window.$message.warning($t("settings.agent.msg.enterKey"));
          return;
        }
        dialogInstance.hide();
        submitAgentSetKey(inputKey.value);
      },
      onClose: () => {
        dialogInstance.hide();
      },
    });
    return;
  }
  submitAgentSetKey(toonflow.inputValues.apiKey);
}

function submitAgentSetKey(key: string) {
  loading.value = true;
  axios
    .post("/setting/agentDeploy/agentSetKey", { key })
    .then(() => {
      window.$message.success($t("settings.agent.msg.configSuccess"));
      getAgentDeploy();
    })
    .catch((err: { message?: string }) => {
      window.$message.error(`${$t("settings.agent.msg.updateConfigFailed")}${err.message}`);
    })
    .finally(() => {
      modelDataShow.value = false;
      loading.value = false;
    });
}

// ── 供应商列表 ──
interface VendorItem {
  id: string; //供应商唯一标识，必须全局唯一
  inputValues: Record<string, string>;
}

const vendorList = ref<VendorItem[]>([]);

// ── 批量切换模型 ──
const batchModelDataShow = ref(false);
const batchSelectValue = ref("");
const batchSelectLabel = ref("");
const batchTesting = ref(false);
const batchTestStatus = ref<"" | "success" | "error">("");
const batchTestMsg = ref("");
const batchApplying = ref(false);
const batchTargetCount = computed(
  () => modelData.value.filter((m) => !m.disabled).length + advancedModelData.value.filter((m) => !m.disabled).length,
);
function openBatchConfig() {
  batchSelectValue.value = "";
  batchSelectLabel.value = "";
  batchTestStatus.value = "";
  batchTestMsg.value = "";
  batchModelDataShow.value = true;
}
async function testBatchModel() {
  if (!batchSelectValue.value) return;
  const [vendorId, modelName] = batchSelectValue.value.split(/:(.+)/);
  if (!vendorId || !modelName) {
    batchTestStatus.value = "error";
    batchTestMsg.value = "未选择有效的模型";
    return;
  }
  batchTesting.value = true;
  batchTestStatus.value = "";
  batchTestMsg.value = "";
  try {
    const res: any = await axios.post("/setting/vendorConfig/modelTest", {
      modelName,
      type: "text",
      id: vendorId,
    });
    batchTestStatus.value = "success";
    const reply = typeof res?.data === "string" ? res.data : JSON.stringify(res?.data);
    batchTestMsg.value = `模型回复（前 200 字）：${(reply || "").slice(0, 200)}`;
  } catch (err: any) {
    batchTestStatus.value = "error";
    batchTestMsg.value = err?.message || String(err);
  } finally {
    batchTesting.value = false;
  }
}
async function confirmBatchConfig() {
  if (!batchSelectValue.value) {
    window.$message.warning("请先选择一个模型");
    return;
  }
  const [vendorId] = batchSelectValue.value.split(/:(.+)/);
  if (!vendorId) {
    window.$message.error("无法解析供应商");
    return;
  }
  const targets = [...modelData.value, ...advancedModelData.value].filter((a) => !a.disabled);
  if (!targets.length) {
    window.$message.warning("没有可配置的 Agent");
    return;
  }
  batchApplying.value = true;
  let ok = 0;
  let fail = 0;
  for (const agent of targets) {
    try {
      await axios.post("/setting/agentDeploy/deployAgentModel", {
        id: agent.id,
        name: agent.name,
        model: batchSelectLabel.value || batchSelectValue.value.split(/:(.+)/)[1],
        modelName: batchSelectValue.value,
        vendorId,
        desc: agent.desc,
        temperature: agent.temperature ?? 1,
        maxOutputTokens: agent.maxOutputTokens ?? 0,
      });
      ok += 1;
    } catch (err: any) {
      fail += 1;
    }
  }
  batchApplying.value = false;
  if (fail === 0) {
    window.$message.success(`已批量配置 ${ok} 个 Agent`);
  } else {
    window.$message.warning(`成功 ${ok} 个，失败 ${fail} 个`);
  }
  await getAgentDeploy();
  batchModelDataShow.value = false;
}

async function getVendorList() {
  try {
    const res = await axios.post("/setting/vendorConfig/getVendorList");
    vendorList.value = res.data.map((item: any) => {
      return {
        ...item,
        enable: item.enable == 1 ? true : false,
      };
    });
  } catch (err: any) {
    window.$message.error(`${$t("settings.vendor.msg.getVendorListFailed")}${err.message}`);
  }
}
//高级配置
const advancedModelData = ref<ModelType[]>([]);
const clearingAdvanced = ref(false);

async function clearAllAdvanced() {
  const targets = advancedModelData.value.filter((a) => !a.disabled);
  if (!targets.length) {
    window.$message.warning("没有可清空的高级 Agent");
    return;
  }
  clearingAdvanced.value = true;
  let ok = 0;
  let fail = 0;
  for (const agent of targets) {
    try {
      await axios.post("/setting/agentDeploy/deployAgentModel", {
        id: agent.id,
        name: agent.name,
        model: "",
        modelName: "",
        vendorId: null,
        desc: agent.desc,
        temperature: 0,
        maxOutputTokens: 0,
      });
      ok += 1;
    } catch {
      fail += 1;
    }
  }
  clearingAdvanced.value = false;
  if (fail === 0) {
    window.$message.success(`已清空 ${ok} 个高级 Agent 配置`);
  } else {
    window.$message.warning(`成功 ${ok} 个，失败 ${fail} 个`);
  }
  await getAgentDeploy();
}
</script>

<style lang="scss" scoped>
.aiConfog {
  display: flex;
  flex-direction: column;
  .banner {
    background-color: var(--td-success-color-focus);
    padding: 16px;
    margin-bottom: 16px;
    border-radius: 4px;
    .content {
      width: 100%;
      .icon {
        color: var(--td-success-color);
        margin-right: 0.5em;
      }
      .btnList {
        .rightBtnList {
          & > * {
            margin-left: 0.5em;
          }
        }
      }
    }
  }
}

.cardGrid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 16px;
  margin-top: 5px;
}

.skillCard {
  cursor: pointer;
  padding: 4px;
  display: flex;
  flex-direction: column;
}

.skillCardHeader {
  display: flex;
  align-items: center;
  justify-content: space-between;
  .headerLeft {
    display: flex;
    align-items: center;
    gap: 8px;
  }
  .skillName {
    font-size: 15px;
    font-weight: 600;
  }
  .fallbackAvatar {
    background: var(--td-brand-color-light);
    color: var(--td-brand-color);
    font-size: 14px;
    font-weight: 600;
  }
}

.skillCardBody {
  font-size: 13px;
  color: var(--td-text-color-secondary);
  line-height: 1.5;
}

.dialogContent {
  padding: 8px 0;
}

.maxTokenRow {
  display: flex;
  align-items: center;
  width: 100%;

  .autoHint {
    flex: 1;
    margin-left: 12px;
    font-size: 13px;
    color: var(--td-text-color-placeholder);
  }
}
</style>
<style lang="scss">
.t-select-option {
  display: block !important;
}
</style>
