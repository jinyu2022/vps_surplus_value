<template>
  <el-main class="vps-calculator-container">
    <h1 class="main-title">VPS 剩余价值计算器</h1>
    <div style="max-width: 1000px; margin: 0 auto">
      <el-row :gutter="20" justify="center">
        <!-- 输入列 -->
        <el-col :xs="24" :sm="12" :md="11">
          <el-card shadow="hover" class="input-card">
            <el-form :model="formData" label-position="top">
              <el-form-item label="参考汇率">
                <el-input :value="formData.referenceRate" disabled>
                  <template #append>
                    <el-tooltip
                      :content="'更新时间: ' + formData.rateUpdateTime"
                      placement="top"
                    >
                      <el-icon><Calendar /></el-icon>
                    </el-tooltip>
                  </template>
                </el-input>
              </el-form-item>

              <el-form-item label="续费金额">
                <el-input
                  v-model="formData.renewalAmount"
                  :precision="2"
                  :step="1"
                  :min="0"
                  :controls="false"
                  style="width: 100%"
                  class="input-with-select"
                >
                  <template #append>
                    <el-select
                      v-model="formData.currency"
                      placeholder="Select"
                      style="width: 120px"
                      @change="updateForeignRate"
                    >
                      <el-option label="美元 (USD)" value="USD" />
                      <el-option label="欧元 (EUR)" value="EUR" />
                      <el-option label="港币 (HKD)" value="HKD" />
                      <el-option label="人民币 (CNY)" value="CNY" />
                      <el-option label="加拿大元 (CAD)" value="CAD" />
                      <el-option label="英镑 (GBP)" value="GBP" />
                      <!-- 可按需添加其他货币 -->
                    </el-select>
                  </template>
                </el-input>
              </el-form-item>

              <el-form-item label="付款周期">
                <el-select
                  v-model="formData.paymentCycle"
                  placeholder="选择周期"
                  style="width: 100%"
                >
                  <el-option label="年付 (Yearly)" value="Yearly" />
                  <el-option label="月付 (Monthly)" value="Monthly" />
                  <el-option label="季付 (Quarterly)" value="Quarterly" />
                  <el-option
                    label="半年付 (Semi-Annually)"
                    value="Semi-Annually"
                  />
                  <el-option label="两年付 (Biennially)" value="Biennially" />
                  <el-option label="三年付 (Triennially)" value="Triennially" />
                  <!-- 可按需添加其他周期 -->
                </el-select>
              </el-form-item>

              <el-form-item label="到期时间">
                <el-config-provider :locale="zhCn">
                  <el-date-picker
                    v-model="formData.expiryDate"
                    :disabled-date="(date) => date < new Date()"
                    type="date"
                    placeholder="选择到期日期"
                    format="YYYY-MM-DD"
                    value-format="YYYY-MM-DD"
                    style="width: 100%"
                  />
                </el-config-provider>
              </el-form-item>

              <el-form-item>
                <el-button
                  type="primary"
                  @click="calculateRemainingValue"
                  :icon="Tickets"
                  style="width: 100%"
                >
                  计算剩余价值
                </el-button>
              </el-form-item>
            </el-form>
          </el-card>
        </el-col>

        <!-- 输出结果列 -->
        <el-col :xs="24" :sm="12" :md="12">
          <el-card shadow="hover" class="output-card">
            <template #header>
              <div class="card-header">
                <span>计算结果</span>
              </div>
            </template>

            <el-descriptions :column="1" border size="large">
              <el-descriptions-item label="交易日期">{{
                results.displayTransactionDate
              }}</el-descriptions-item>
              <el-descriptions-item label="外币汇率">{{
                results.displayForeignRate.toFixed(4)
              }}</el-descriptions-item>
              <el-descriptions-item label="续费价格"
                >{{ results.renewalPrice }} {{ formData.currency }} /
                {{ getPaymentCycleText() }}
              </el-descriptions-item>
              <el-descriptions-item label="剩余天数">
                <el-tag
                  :type="results.remainingDays > 30 ? 'success' : 'danger'"
                >
                  {{ results.remainingDays }} 天
                </el-tag>
                （于 {{ results.expiryStatus }} 过期）
              </el-descriptions-item>
              <el-descriptions-item label="剩余价值">
                <strong style="color: #e6a23c"
                  >{{ results.remainingValue.toFixed(2) }} 元</strong
                >
              </el-descriptions-item>
            </el-descriptions>

            <div class="share-section">
              <el-text type="info" size="small">图片分享链接</el-text>
              <div class="share-buttons">
                <el-button :icon="View" @click="viewLink" :disabled="!shareLink"
                  >查看</el-button
                >
                <el-button
                  :icon="CopyDocument"
                  @click="copyLink"
                  :disabled="!shareLink"
                  >复制</el-button
                >
              </div>
            </div>
          </el-card>
        </el-col>
      </el-row>
    </div>
    <!-- <footer class="app-footer">
        本项目开源于
        <el-link type="primary" href="https://github.com" target="_blank"
          ><el-icon><Link /></el-icon> GitHub</el-link
        >
        | <el-link type="primary">关于</el-link> |
        <el-link type="primary">更多</el-link>
      </footer> -->
  </el-main>
</template>

<script setup>
import { ref, reactive, computed } from "vue";
import {
  ElMessage,
  ElMain,
  ElRow,
  ElCol,
  ElCard,
  ElForm,
  ElFormItem,
  ElInput,
  ElSelect,
  ElOption,
  ElDatePicker,
  ElButton,
  ElDescriptions,
  ElDescriptionsItem,
  ElTag,
  ElText,
  ElLink,
  ElIcon,
  ElTooltip,
  ElConfigProvider,
} from "element-plus";
import zhCn from "element-plus/es/locale/lang/zh-cn";
import {
  Calendar,
  Tickets,
  View,
  CopyDocument,
  Link,
} from "@element-plus/icons-vue";

// --- 响应式数据 ---
const date = new Date();
const options = {
  year: "numeric",
  month: "2-digit",
  day: "2-digit",
  hour: "numeric",
};
const currentDate = date.toLocaleDateString("zh-CN", options);

/**
 * 汇率数据
 * 存储从API获取的所有汇率数据
 */
const exchangeRates = reactive({
  rates: {},
  lastUpdated: "",
  nextUpdate: "",
  baseCode: "CNY",
});

/**
 * 表单数据对象
 * 包含用户输入的所有数据
 */
const formData = reactive({
  referenceRate: 7.267, // 参考汇率值
  rateUpdateTime: "尚未更新", // 汇率更新时间
  renewalAmount: "12", // 续费金额
  currency: "USD", // 货币类型，默认美元
  paymentCycle: "Yearly", // 付款周期，默认年付
  expiryDate: new Date(Date.now() + 20 * 24 * 60 * 60 * 1000)
    .toLocaleDateString("zh-CN", {
      year: "numeric",
      month: "2-digit",
      day: "2-digit",
    })
    .replace(/\//g, "-"), // 到期日期
});

/**
 * 计算结果对象
 * 存储计算后展示的所有值
 */
const results = reactive({
  displayTransactionDate: currentDate, // 显示的交易日期
  displayForeignRate: computed(() => formData.referenceRate), // 显示的外币汇率
  renewalPrice: computed(() => parseFloat(formData.renewalAmount)), // 续费价格
  remainingDays: 0, // 剩余天数
  expiryStatus: computed(() => formData.expiryDate), // 到期日期
  remainingValue: 0.0, // 剩余价值（人民币）
  totalValue: 0.0, // 总价值（人民币）
});

const shareLink = ref(""); // 用于存储分享链接
/**
 * 获取最新汇率数据
 */
const fetchExchangeRates = async () => {
  try {
    const response = await fetch("https://open.er-api.com/v6/latest/CNY");
    const data = await response.json();

    if (data.result === "success") {
      exchangeRates.rates = data.rates;
      exchangeRates.lastUpdated = data.time_last_update_utc;
      exchangeRates.nextUpdate = data.time_next_update_utc;
      exchangeRates.baseCode = data.base_code;

      // 格式化更新时间为本地时间格式
      const updateDate = new Date(data.time_last_update_unix * 1000);
      formData.rateUpdateTime = updateDate.toLocaleString("zh-CN");

      // 更新当前选择的货币汇率
      updateForeignRate();

      ElMessage.success("汇率数据已更新");
    } else {
      ElMessage.error("获取汇率数据失败");
    }
  } catch (error) {
    console.error("获取汇率出错:", error);
    ElMessage.error("获取汇率数据失败，请检查网络连接");
  }
};

/**
 * 根据选择的货币更新汇率
 */
const updateForeignRate = () => {
  if (exchangeRates.rates && Object.keys(exchangeRates.rates).length > 0) {
    // 如果是CNY，汇率为1，否则使用API提供的汇率的倒数
    if (formData.currency === "CNY") {
      formData.referenceRate = 1;
    } else {
      // 从CNY到其他货币的汇率是其他货币到CNY的倒数
      const rate = exchangeRates.rates[formData.currency];
      if (rate) {
        // 精确计算倒数，保留四位小数
        formData.referenceRate = parseFloat((1 / rate).toFixed(4));
      }
    }
  }
};

/**
 * 获取周期对应的准确天数
 * @param {string} cycle - 付款周期
 * @param {Date} startDate - 开始日期，默认为当前日期
 * @returns {number} - 周期对应的精确天数
 */
const getDaysInCycle = (cycle, startDate = new Date()) => {
  // 创建开始日期的副本
  const start = new Date(startDate);
  // 结束日期初始化为与开始日期相同
  const end = new Date(startDate);

  // 根据不同周期调整结束日期
  switch (cycle) {
    case "Monthly":
      // 增加一个月
      end.setMonth(end.getMonth() + 1);
      break;
    case "Quarterly":
      // 增加三个月
      end.setMonth(end.getMonth() + 3);
      break;
    case "Semi-Annually":
      // 增加六个月
      end.setMonth(end.getMonth() + 6);
      break;
    case "Yearly":
      // 增加一年
      end.setFullYear(end.getFullYear() + 1);
      break;
    case "Biennially":
      // 增加两年
      end.setFullYear(end.getFullYear() + 2);
      break;
    case "Triennially":
      // 增加三年
      end.setFullYear(end.getFullYear() + 3);
      break;
    default:
      // 默认增加一年
      end.setFullYear(end.getFullYear() + 1);
  }

  // 计算两个日期之间的毫秒差
  const diffTime = end.getTime() - start.getTime();
  // 将毫秒转换为天数并四舍五入
  return Math.round(diffTime / (1000 * 60 * 60 * 24));
};

/**
 * 获取付款周期的中文显示文本
 * @returns {string} - 周期的中文文本
 */
const getPaymentCycleText = () => {
  switch (formData.paymentCycle) {
    case "Monthly":
      return "月";
    case "Quarterly":
      return "季度";
    case "Semi-Annually":
      return "半年";
    case "Yearly":
      return "年";
    case "Biennially":
      return "两年";
    case "Triennially":
      return "三年";
    default:
      return "周期";
  }
};

/**
 * 计算剩余价值的主函数
 * 根据用户输入计算并更新结果
 */
const calculateRemainingValue = () => {
  // 基本输入验证
  if (
    typeof results.renewalPrice !== "number" ||
    !formData.expiryDate ||
    results.renewalPrice < 0
  ) {
    ElMessage.error("请确保所有必填项已正确填写，且金额为正数。");
    return;
  }
  const expiryDate = new Date(formData.expiryDate); // 到期日期
  if (isNaN(expiryDate.getTime())) {
    ElMessage.error("输入的到期日期无效，请检查格式并重新输入。");
    return;
  }
  const today = new Date(); // 当前日期，用于比较

  // 确保日期有效
  if (isNaN(expiryDate.getTime())) {
    ElMessage.error("输入的日期格式无效，请使用 YYYY-MM-DD 格式。");
    return;
  }

  // --- 开始计算 ---

  // 计算订阅周期的总价值（人民币）
  const costPerCycleCNY =
    formData.currency === "CNY"
      ? parseFloat(results.renewalPrice)
      : parseFloat(results.renewalPrice) * formData.referenceRate;
  const daysInCycle = getDaysInCycle(formData.paymentCycle); // 获取周期天数
  const dailyValueCNY = daysInCycle > 0 ? costPerCycleCNY / daysInCycle : 0; // 计算每天的价值
  results.totalValue = costPerCycleCNY; // 当前周期支付的总价值

  // 调整到期日为当天的结束（23:59:59）以便更精确计算
  const expiryEndOfDay = new Date(expiryDate);
  expiryEndOfDay.setHours(23, 59, 59, 999);
  const timeDiffPrecise = expiryEndOfDay.getTime() - today.getTime();

  // 使用Math.ceil向上取整包含最后一天，确保至少为0
  results.remainingDays = Math.max(
    0,
    Math.ceil(timeDiffPrecise / (1000 * 60 * 60 * 24))
  );

  // 计算剩余价值
  results.remainingValue = Math.max(0, dailyValueCNY * results.remainingDays);

  // 更新显示结果
  results.displayTransactionDate = currentDate;

  uploadImage(); // 计算完成后上传图片
};

const uploadImage = async () => {
  const svg = `
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 570 400" style="max-width: 1004px; width: 100%; height: auto;" width="1004">
  <defs>
    <linearGradient id="lightBlueGradient" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" style="stop-color:#93ccea;stop-opacity:1"></stop>
      <stop offset="100%" style="stop-color:#7bebcf;stop-opacity:1"></stop>
    </linearGradient>
    <linearGradient id="tableGradient" x1="0%" y1="0%" x2="0%" y2="100%">
      <stop offset="0%" style="stop-color:rgba(255,255,255,0.4);stop-opacity:1"></stop>
      <stop offset="100%" style="stop-color:rgba(255,255,255,0.3);stop-opacity:1"></stop>
    </linearGradient>
    <filter id="shadowEffect" x="-5%" y="-5%" width="110%" height="110%">
      <feGaussianBlur in="SourceAlpha" stdDeviation="2"></feGaussianBlur>
      <feOffset dx="1" dy="1" result="offsetBlur"></feOffset>
      <feComponentTransfer>
        <feFuncA type="linear" slope="0.2"></feFuncA>
      </feComponentTransfer>
      <feMerge>
        <feMergeNode></feMergeNode>
        <feMergeNode in="SourceGraphic"></feMergeNode>
      </feMerge>
    </filter>
  </defs>
  <rect x="0" y="0" width="570" height="400" rx="12" fill="url(#lightBlueGradient)"></rect>
  <circle cx="40" cy="40" r="15" fill="rgba(255,255,255,0.15)"></circle>
  <circle cx="530" cy="360" r="15" fill="rgba(255,255,255,0.15)"></circle>
  <path d="M40,40 L530,360" stroke="rgba(255,255,255,0.25)" stroke-width="1" stroke-dasharray="5,8"></path>
  <rect x="55" y="25" width="460" height="335" rx="12" fill="rgba(255, 255, 255, 0.35)" filter="url(#shadowEffect)"></rect>
  <text x="285" y="75" font-family="'Segoe UI', Roboto, Arial, sans-serif" font-size="24" font-weight="600" fill="#1a365d" text-anchor="middle">VPS 剩余价值计算器</text>
  <rect x="110" y="110" width="350" height="200" rx="8" fill="url(#tableGradient)" stroke="rgba(255,255,255,0.7)" stroke-width="1"></rect>
  <rect x="110" y="110" width="350" height="40" rx="8 8 0 0" fill="rgba(173, 216, 230, 0.3)" stroke="none"></rect>
  <rect x="110" y="150" width="350" height="40" fill="rgba(255,255,255,0.3)" stroke="none"></rect>
  <rect x="110" y="190" width="350" height="40" fill="rgba(173, 216, 230, 0.3)" stroke="none"></rect>
  <rect x="110" y="230" width="350" height="40" fill="rgba(255,255,255,0.3)" stroke="none"></rect>
  <rect x="110" y="270" width="350" height="40" rx="0 0 8 8" fill="rgba(173, 216, 230, 0.3)" stroke="none"></rect>
  <line x1="200" y1="110" x2="200" y2="310" stroke="rgba(112, 161, 215, 0.6)" stroke-width="1"></line>
  <g font-family="'Segoe UI', Roboto, Arial, sans-serif" font-size="16">
    <text x="120" y="135" fill="#1a365d" _mstTextHash="20372209" _mstHash="199">交易日期：</text>
    <text x="220" y="135" fill="#1a365d" font-weight="500" _mstTextHash="75270" _mstHash="198">${
      results.displayTransactionDate
    }</text>
    <text x="120" y="175" fill="#1a365d" _mstTextHash="21004737" _mstHash="197">外币汇率：</text>
    <text x="220" y="175" fill="#1a365d" font-weight="500" _mstTextHash="27963" _mstHash="196">${
      results.displayForeignRate
    }</text>
    <text x="120" y="215" fill="#1a365d" _mstTextHash="21889608" _mstHash="195">续费价格：</text>
    <text x="220" y="215" fill="#1a365d" font-weight="500" _mstTextHash="17265989" _mstHash="194">${
      results.renewalPrice
    } ${formData.currency} / ${getPaymentCycleText()}
    </text>
    <text x="120" y="255" fill="#1a365d" _mstTextHash="19417502" _mstHash="193">剩余天数：</text>
    <text x="220" y="255" fill="${
      results.remainingDays > 30 ? "#00875A" : "#FF8C00"
    }" font-weight="bold" _mstTextHash="2378259" _mstHash="192">
      ${results.remainingDays} 天
    </text>
    <text x="270" y="255" font-size="14" fill="#4a5568" _mstTextHash="18147753" _mstHash="191">
      （于 ${results.expiryStatus} 到期）
    </text>
    <text x="120" y="295" fill="#1a365d" _mstTextHash="18406492" _mstHash="190">剩余价值：</text>
    <text x="220" y="295" fill="#2B6CB0" font-weight="bold" _mstTextHash="3273959" _mstHash="189">
      ${results.remainingValue.toFixed(2)} 元
    </text>
    <text x="290" y="295" font-size="14" fill="#4a5568" _mstTextHash="6673940" _mstHash="188">
      （总 ${results.totalValue.toFixed(2)} 元）
    </text>
  </g>
  <path d="M170,350 Q285,370 400,350" stroke="rgba(112, 161, 215, 0.5)" stroke-width="1" fill="none"></path>
</svg>`;

  const blob = new Blob([svg], { type: "image/svg+xml" });
  // 创建FormData对象
  const formDataImg = new FormData();
  formDataImg.append("file", blob, "vps_calculator.svg");

  try {
    const response = await fetch(
      `https://proxy.corsfix.com/?https://skyimg.net/api/upload`,
      {
        method: "POST",
        body: formDataImg,
      }
    );
    if (response.ok) {
      const data = await response.json();
      console.log(data);
      if (Array.isArray(data) && data[0] && data[0].url) {
        shareLink.value = data[0].url; // 确保返回的JSON中有一个url字段
        ElMessage.success("图片上传成功！");
      } else {
        throw new Error("API响应格式无效，未找到图片链接");
      }
    } else {
      ElMessage.error("图片上传失败，请重试。");
    }
  } catch (error) {
    ElMessage.error("上传过程中发生错误: " + error.message);
  }
};
/**
 * 查看分享链接
 */
const viewLink = () => {
  if (!shareLink.value) {
    ElMessage.error("请先上传图片以获取分享链接。");
    return;
  }
  window.open(shareLink.value, "_blank");
};

/**
 * 复制分享链接
 */
const copyLink = () => {
  if (!shareLink.value) {
    ElMessage.error("请先上传图片以获取分享链接。");
    return;
  }
  if (navigator.clipboard && navigator.clipboard.writeText) {
    navigator.clipboard
      .writeText(`![image](${shareLink.value})`)
      .then(() => {
        ElMessage.success("复制成功！");
      })
      .catch((err) => {
        ElMessage.error("复制失败: ", err);
      });
  } else {
    // Fallback for browsers without Clipboard API support
    const tempInput = document.createElement("input");
    tempInput.value = `![image](${shareLink.value})`;
    document.body.appendChild(tempInput);
    tempInput.select();
    try {
      document.execCommand("copy");
      ElMessage.success("复制成功！");
    } catch (err) {
      ElMessage.error("复制失败: ", err);
    }
    document.body.removeChild(tempInput);
  }
};

// 组件挂载时获取汇率数据

fetchExchangeRates();
</script>

<style scoped>
.vps-calculator-container {
  padding: 20px;
  background: linear-gradient(
    135deg,
    #f5f7fa 0%,
    #c3cfe2 100%
  ); /* Subtle gradient background */
  min-height: 100vh; /* Ensure it fills viewport height */
}

.main-title {
  text-align: center;
  margin-bottom: 30px;
  font-size: 2.5em; /* Make title larger */
  font-weight: bold;
  color: transparent; /* Make text transparent */
  background: linear-gradient(
    90deg,
    #ff8a00,
    #e52e71
  ); /* Gradient background */
  -webkit-background-clip: text; /* Clip background to text */
  background-clip: text; /* Standard property */
  display: inline-block; /* Needed for background clipping */
  padding-bottom: 5px; /* Add some space below gradient */
}

.input-card,
.output-card {
  background-color: rgba(255, 255, 255, 0.85); /* Slightly transparent cards */
  backdrop-filter: blur(5px); /* Frosted glass effect */
  border-radius: 15px;
  padding: 20px; /* Add internal padding */
  margin-bottom: 20px; /* Space between cards on smaller screens */
}

/* Enhance card headers (optional) */
.card-header {
  font-weight: bold;
  font-size: 1.1em;
}

/* Ensure input number controls are visible */
.el-input-number .el-input__inner {
  text-align: left !important; /* Align number input text left */
}

/* Fix alignment for input with select append */
.input-with-select .el-input-group__append .el-select .el-input__inner {
  padding-left: 10px; /* Adjust padding if needed */
  padding-right: 30px; /* Ensure space for dropdown arrow */
}
.input-with-select .el-input-group__append {
  background-color: var(--el-fill-color-blank); /* Match background */
}

.share-section {
  margin-top: 25px;
  padding-top: 15px;
  border-top: 1px solid var(--el-border-color-lighter);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.share-buttons .el-button {
  margin-left: 10px;
}

.app-footer {
  text-align: center;
  margin-top: 40px;
  color: #909399;
  font-size: 0.9em;
}

.app-footer .el-link {
  margin: 0 5px;
  vertical-align: baseline; /* Align link text better */
}

/* Center title container */
h1.main-title {
  display: block; /* Make it block to center */
  text-align: center;
}

/* Add some responsiveness adjustments */
@media (max-width: 768px) {
  .main-title {
    font-size: 2em;
  }
  .el-col {
    margin-bottom: 20px; /* Add space between columns when stacked */
  }
  .share-section {
    flex-direction: column;
    align-items: flex-start;
  }
  .share-buttons {
    margin-top: 10px;
    margin-left: -10px; /* Counteract button margin */
  }
}
</style>
