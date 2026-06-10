<script setup>
import { computed, onMounted, reactive, ref } from "vue";
import { parkingApi } from "../api/parking";
import StatusBadge from "../components/StatusBadge.vue";

const spaces = ref([]);
const orders = ref([]);
const message = ref("");
const quote = ref(null);
const showSettleDialog = ref(false);
const currentOrder = ref(null);
const settleForm = reactive({
  exit_time: new Date().toISOString().slice(0, 16),
  remark: "",
  operator: "",
});
const entryForm = reactive({
  plate_number: "",
  space_code: "",
});
const calcForm = reactive({
  entry_time: new Date(Date.now() - 90 * 60 * 1000).toISOString().slice(0, 16),
  exit_time: new Date().toISOString().slice(0, 16),
});

const freeSpaces = computed(() => spaces.value.filter((space) => ["free", "reserved"].includes(space.status)));
const parkingOrders = computed(() => orders.value.filter((order) => order.status === "parking"));
const historyOrders = computed(() => orders.value.filter((order) => order.status === "paid"));

async function loadData() {
  const [spaceData, orderData] = await Promise.all([parkingApi.getSpaces(), parkingApi.getOrders()]);
  spaces.value = spaceData.items;
  orders.value = orderData.items;
  if (!entryForm.space_code && freeSpaces.value[0]) entryForm.space_code = freeSpaces.value[0].code;
}

async function createEntry() {
  message.value = "";
  try {
    await parkingApi.entry(entryForm);
    entryForm.plate_number = "";
    await loadData();
    message.value = "入场登记成功";
  } catch (err) {
    message.value = err.message;
  }
}

async function calculate() {
  quote.value = await parkingApi.calculate(calcForm);
}

function openSettleDialog(order) {
  currentOrder.value = order;
  settleForm.exit_time = new Date().toISOString().slice(0, 16);
  settleForm.remark = "";
  settleForm.operator = "";
  showSettleDialog.value = true;
}

async function confirmSettle() {
  if (!settleForm.operator) {
    message.value = "请输入操作员姓名";
    return;
  }
  const result = await parkingApi.exit(currentOrder.value.id, {
    exit_time: settleForm.exit_time,
    remark: settleForm.remark || null,
    operator: settleForm.operator,
  });
  message.value = `${currentOrder.value.plate_number} 已结算，金额 ¥${result.amount}`;
  showSettleDialog.value = false;
  currentOrder.value = null;
  await loadData();
}

function closeSettleDialog() {
  showSettleDialog.value = false;
  currentOrder.value = null;
}

onMounted(loadData);
</script>

<template>
  <div class="page-stack">
    <header class="page-header">
      <div>
        <h2>临时停车计费</h2>
        <p>登记入场、试算费用并完成离场结算。</p>
      </div>
    </header>

    <div class="billing-grid">
      <form class="form-panel" @submit.prevent="createEntry">
        <h3>车辆入场</h3>
        <label>车牌号<input v-model="entryForm.plate_number" required /></label>
        <label>
          车位
          <select v-model="entryForm.space_code" required>
            <option v-for="space in freeSpaces" :key="space.id" :value="space.code">{{ space.code }}</option>
          </select>
        </label>
        <button class="primary-button" type="submit">登记入场</button>
        <p v-if="message" class="hint-text">{{ message }}</p>
      </form>

      <form class="form-panel" @submit.prevent="calculate">
        <h3>费用试算</h3>
        <label>入场时间<input v-model="calcForm.entry_time" type="datetime-local" required /></label>
        <label>离场时间<input v-model="calcForm.exit_time" type="datetime-local" required /></label>
        <button class="secondary-button" type="submit">计算费用</button>
        <p v-if="quote" class="quote-text">停车 {{ quote.duration_hours }} 小时，应收 ¥{{ quote.amount }}</p>
      </form>
    </div>

    <section class="table-section">
      <h3>在场车辆</h3>
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>订单</th>
              <th>车牌</th>
              <th>车位</th>
              <th>入场时间</th>
              <th>状态</th>
              <th>操作</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="order in parkingOrders" :key="order.id">
              <td>#{{ order.id }}</td>
              <td>{{ order.plate_number }}</td>
              <td>{{ order.space_code }}</td>
              <td>{{ order.entry_time }}</td>
              <td><StatusBadge :status="order.status" /></td>
              <td><button class="small-button" type="button" @click="openSettleDialog(order)">离场结算</button></td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

    <section class="table-section">
      <h3>历史结算记录</h3>
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>订单</th>
              <th>车牌</th>
              <th>车位</th>
              <th>入场时间</th>
              <th>离场时间</th>
              <th>时长(小时)</th>
              <th>金额</th>
              <th>操作员</th>
              <th>备注</th>
              <th>状态</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="order in historyOrders" :key="order.id">
              <td>#{{ order.id }}</td>
              <td>{{ order.plate_number }}</td>
              <td>{{ order.space_code }}</td>
              <td>{{ order.entry_time }}</td>
              <td>{{ order.exit_time }}</td>
              <td>{{ order.duration_hours }}</td>
              <td>¥{{ order.amount }}</td>
              <td>{{ order.operator || '-' }}</td>
              <td>{{ order.remark || '-' }}</td>
              <td><StatusBadge :status="order.status" /></td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

    <div v-if="showSettleDialog" class="modal-overlay" @click.self="closeSettleDialog">
      <div class="modal-content">
        <h3>离场结算</h3>
        <div class="order-info" v-if="currentOrder">
          <p><strong>车牌：</strong>{{ currentOrder.plate_number }}</p>
          <p><strong>车位：</strong>{{ currentOrder.space_code }}</p>
          <p><strong>入场时间：</strong>{{ currentOrder.entry_time }}</p>
        </div>
        <form @submit.prevent="confirmSettle">
          <label>
            离场时间
            <input v-model="settleForm.exit_time" type="datetime-local" required />
          </label>
          <label>
            操作员 <span class="required">*</span>
            <input v-model="settleForm.operator" placeholder="请输入操作员姓名" required />
          </label>
          <label>
            备注
            <textarea v-model="settleForm.remark" placeholder="请输入备注信息（可选）" rows="3"></textarea>
          </label>
          <div class="modal-actions">
            <button type="button" class="secondary-button" @click="closeSettleDialog">取消</button>
            <button type="submit" class="primary-button">确认结算</button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>
