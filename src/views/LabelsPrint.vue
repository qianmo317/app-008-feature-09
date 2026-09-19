<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, nextTick } from 'vue';
import { useRoute } from 'vue-router';
import { getTask, saveTask } from '../db';
import { generateQRDataURL, formatDateTime } from '../utils';
import type { MoveTask, Box } from '../types';

type TimeFilter = 'all' | 'today' | '7d';

const route = useRoute();
const taskId = route.params.id as string;

const task = ref<MoveTask | null>(null);

// 筛选条件
const codeQuery = ref('');
const roomFilter = ref('');
const timeFilter = ref<TimeFilter>('all');

// 勾选的箱子 id
const selectedIds = ref<Set<string>>(new Set());

// 实际送入打印区的标签（仅打印时填充，避免一进页面就整份渲染）
const printItems = ref<{ box: Box; qr: string }[]>([]);
const printing = ref(false);

const qrCache = new Map<string, string>();

// 同一批标签连点防护：相同集合在该时间窗内只放行一次
const DOUBLE_CLICK_GUARD_MS = 2000;
let lastPrintKey = '';
let lastPrintAt = 0;

// 当前打印任务，afterprint 后用于计数
let currentJob: Box[] | null = null;
let finishHandled = false;
let fallbackTimer: number | null = null;

const filteredBoxes = computed(() => {
  if (!task.value) return [];
  const q = codeQuery.value.trim().toLowerCase();
  const now = Date.now();
  const today = new Date();
  return task.value.boxes
    .filter((b) => {
      if (q && !b.code.toLowerCase().includes(q)) return false;
      if (roomFilter.value && b.roomTo !== roomFilter.value) return false;
      if (timeFilter.value === '7d') {
        if (now - b.createdAt > 7 * 86400000) return false;
      } else if (timeFilter.value === 'today') {
        const d = new Date(b.createdAt);
        if (
          d.getFullYear() !== today.getFullYear() ||
          d.getMonth() !== today.getMonth() ||
          d.getDate() !== today.getDate()
        ) {
          return false;
        }
      }
      return true;
    })
    .sort((a, b) => a.code.localeCompare(b.code));
});

const filteredSelectedBoxes = computed(() =>
  filteredBoxes.value.filter((b) => selectedIds.value.has(b.id))
);

const allFilteredSelected = computed(
  () =>
    filteredBoxes.value.length > 0 &&
    filteredBoxes.value.every((b) => selectedIds.value.has(b.id))
);

const stats = computed(() => {
  const boxes = task.value?.boxes ?? [];
  const printed = boxes.filter((b) => (b.printCount ?? 0) > 0).length;
  const reprinted = boxes.filter((b) => (b.printCount ?? 0) >= 2).length;
  return {
    total: boxes.length,
    printed,
    unprinted: boxes.length - printed,
    reprinted,
  };
});

async function load() {
  const t = await getTask(taskId);
  task.value = t;
  if (!t) return;
  // 清掉已不存在（被删除）箱子的勾选
  const ids = new Set(t.boxes.map((b) => b.id));
  selectedIds.value = new Set([...selectedIds.value].filter((id) => ids.has(id)));
}

async function ensureQR(box: Box): Promise<string> {
  const cached = qrCache.get(box.id);
  if (cached) return cached;
  const qr = await generateQRDataURL(taskId, box.code);
  qrCache.set(box.id, qr);
  return qr;
}

function toggleOne(id: string) {
  const next = new Set(selectedIds.value);
  if (next.has(id)) next.delete(id);
  else next.add(id);
  selectedIds.value = next;
}

function toggleAllFiltered() {
  const next = new Set(selectedIds.value);
  if (allFilteredSelected.value) {
    filteredBoxes.value.forEach((b) => next.delete(b.id));
  } else {
    filteredBoxes.value.forEach((b) => next.add(b.id));
  }
  selectedIds.value = next;
}

function clearSelection() {
  selectedIds.value = new Set();
}

async function printBoxes(targets: Box[]) {
  // 打印进行中直接忽略，杜绝连点打出两份
  if (printing.value || targets.length === 0) return;
  const key = [...targets.map((b) => b.id)].sort().join(',');
  const now = Date.now();
  if (key === lastPrintKey && now - lastPrintAt < DOUBLE_CLICK_GUARD_MS) return;
  lastPrintKey = key;
  lastPrintAt = now;

  printing.value = true;
  const built = await Promise.all(
    targets.map(async (box) => ({ box, qr: await ensureQR(box) }))
  );
  currentJob = targets;
  finishHandled = false;
  printItems.value = built;
  window.addEventListener('afterprint', onAfterPrint);

  await nextTick();
  window.print();
  // window.print() 在多数浏览器上同步阻塞，定时器在弹窗关闭后才开始走，
  // 作为 afterprint 不触发时（如旧版 Safari）的兜底
  fallbackTimer = window.setTimeout(onAfterPrint, 1000);
}

function onAfterPrint() {
  if (finishHandled) return;
  finishHandled = true;
  window.removeEventListener('afterprint', onAfterPrint);
  if (fallbackTimer !== null) {
    clearTimeout(fallbackTimer);
    fallbackTimer = null;
  }
  finishPrint();
}

async function finishPrint() {
  const targets = currentJob;
  currentJob = null;
  printing.value = false;
  printItems.value = [];

  if (targets && targets.length > 0) {
    // 重新取数后再写，避免覆盖打印弹窗期间箱子的状态变更或删除
    const fresh = await getTask(taskId);
    if (fresh) {
      const now = Date.now();
      const targetIds = new Set(targets.map((b) => b.id));
      let changed = false;
      fresh.boxes.forEach((b) => {
        if (targetIds.has(b.id)) {
          b.printCount = (b.printCount ?? 0) + 1;
          b.lastPrintedAt = now;
          changed = true;
        }
      });
      if (changed) await saveTask(fresh);
      // 已成功送印的取消勾选
      selectedIds.value = new Set(
        [...selectedIds.value].filter((id) => !targetIds.has(id))
      );
    }
  }
  await load();
}

function onVisible() {
  // 从箱子详情页删除箱子后返回时，重新拉取，删除的箱子不再出现
  if (document.visibilityState === 'visible' && !currentJob) load();
}

onMounted(() => {
  load();
  document.addEventListener('visibilitychange', onVisible);
});

onUnmounted(() => {
  document.removeEventListener('visibilitychange', onVisible);
  window.removeEventListener('afterprint', onAfterPrint);
  if (fallbackTimer !== null) clearTimeout(fallbackTimer);
});
</script>

<template>
  <div v-if="task">
    <div class="header no-print">
      <router-link :to="`/task/${task.id}`" class="back">←</router-link>
      <h1>标签打印</h1>
    </div>

    <div class="page no-print">
      <!-- 打印情况总览 -->
      <div class="summary card">
        <div><strong>{{ stats.total }}</strong><span>总箱数</span></div>
        <div><strong style="color:var(--text-secondary)">{{ stats.unprinted }}</strong><span>未打印</span></div>
        <div><strong style="color:var(--info)">{{ stats.printed }}</strong><span>已打印</span></div>
        <div><strong :style="{color: stats.reprinted ? 'var(--danger)' : undefined}">{{ stats.reprinted }}</strong><span>补打≥2次</span></div>
      </div>

      <!-- 筛选 -->
      <div class="card">
        <label class="label">按箱号查找</label>
        <input v-model="codeQuery" class="input" placeholder="输入箱号，如 A-014" />
        <div class="filter-row">
          <select v-model="roomFilter" class="select">
            <option value="">全部房间</option>
            <option v-for="r in task.rooms" :key="r" :value="r">{{ r }}</option>
          </select>
          <select v-model="timeFilter" class="select">
            <option value="all">封箱：全部时间</option>
            <option value="today">今天封箱</option>
            <option value="7d">近 7 天封箱</option>
          </select>
        </div>
      </div>

      <div v-if="task.boxes.length === 0" class="empty">
        还没有已封箱的箱子，先去封箱登记吧
      </div>

      <template v-else>
        <div class="select-bar">
          <label class="check-all">
            <input type="checkbox" :checked="allFilteredSelected" @change="toggleAllFiltered" />
            全选当前结果（{{ filteredBoxes.length }}）
          </label>
          <button v-if="selectedIds.size > 0" class="link-btn" @click="clearSelection">
            清空勾选（{{ selectedIds.size }}）
          </button>
        </div>

        <div v-if="filteredBoxes.length === 0" class="empty">
          没有符合筛选条件的箱子
        </div>

        <div
          v-for="b in filteredBoxes"
          :key="b.id"
          class="box-row card"
          :class="{ reprint: (b.printCount ?? 0) >= 2, selected: selectedIds.has(b.id) }"
        >
          <input
            type="checkbox"
            class="row-check"
            :checked="selectedIds.has(b.id)"
            @change="toggleOne(b.id)"
          />
          <div class="row-main" @click="toggleOne(b.id)">
            <div class="row-top">
              <strong>{{ b.code }}</strong>
              <span class="room">{{ b.roomTo }}</span>
              <span v-if="b.fragile" class="mark fragile">易碎</span>
              <span v-if="b.liquid" class="mark liquid">液体</span>
            </div>
            <div class="row-sub">
              封箱 {{ formatDateTime(b.createdAt) }}<template v-if="b.tags.length"> · {{ b.tags.join('、') }}</template>
            </div>
            <div class="row-status">
              <span v-if="(b.printCount ?? 0) === 0" class="badge badge-none">未打印</span>
              <span v-else-if="(b.printCount ?? 0) === 1" class="badge badge-once">
                已打印 1 次 · {{ formatDateTime(b.lastPrintedAt!) }}
              </span>
              <span v-else class="badge badge-multi">
                ⚠ 已打印 {{ b.printCount }} 次 · 最近 {{ formatDateTime(b.lastPrintedAt!) }}
              </span>
            </div>
          </div>
          <button
            class="btn btn-secondary row-print"
            :disabled="printing"
            @click.stop="printBoxes([b])"
          >
            {{ (b.printCount ?? 0) > 0 ? '补打' : '打印此张' }}
          </button>
        </div>

        <div class="bottom-bar">
          <button
            class="btn btn-block"
            :disabled="printing || filteredSelectedBoxes.length === 0"
            @click="printBoxes(filteredSelectedBoxes)"
          >
            打印选中{{ filteredSelectedBoxes.length ? `（${filteredSelectedBoxes.length} 张）` : '' }}
          </button>
          <button
            class="btn btn-secondary btn-block"
            :disabled="printing || filteredBoxes.length === 0"
            @click="printBoxes(filteredBoxes)"
          >
            打印全部筛选结果（{{ filteredBoxes.length }} 张）
          </button>
          <div v-if="printing" class="printing-hint">正在调起打印，请勿重复点击…</div>
        </div>
      </template>
    </div>

    <!-- 打印区：只包含本次选中的标签 -->
    <div id="print-only">
      <div class="print-grid">
        <div v-for="it in printItems" :key="it.box.id" class="print-label">
          <div class="print-code">{{ it.box.code }}</div>
          <div class="print-room">{{ it.box.roomTo }}</div>
          <img class="print-qr" :src="it.qr" />
          <div class="print-tags">{{ it.box.tags.join(' · ') }}</div>
          <div class="print-warn">
            {{ it.box.fragile ? '易碎' : '' }} {{ it.box.liquid ? '液体禁运' : '' }}
          </div>
          <div v-if="(it.box.printCount ?? 0) > 0" class="print-count">
            第 {{ (it.box.printCount ?? 0) + 1 }} 次打印（补打）
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.summary {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  text-align: center;
  gap: 6px;
}
.summary strong {
  display: block;
  font-size: 22px;
}
.summary span {
  font-size: 12px;
  color: var(--text-secondary);
}

.filter-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
  margin-top: 8px;
}

.select-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin: 4px 2px 10px;
  font-size: 14px;
}
.check-all {
  display: flex;
  align-items: center;
  gap: 6px;
  cursor: pointer;
  user-select: none;
}
.link-btn {
  border: none;
  background: none;
  color: var(--primary-dark);
  font-size: 13px;
  cursor: pointer;
  padding: 4px;
}

.box-row {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 12px;
  margin-bottom: 10px;
}
.box-row.selected {
  border-color: var(--primary);
  box-shadow: 0 0 0 2px rgba(245, 158, 11, 0.25);
}
/* 反复补打的箱子在页面上醒目标出 */
.box-row.reprint {
  border-color: var(--danger);
  background: color-mix(in srgb, var(--danger) 8%, var(--surface));
}
.row-check {
  width: 20px;
  height: 20px;
  flex-shrink: 0;
  accent-color: var(--primary);
}
.row-main {
  flex: 1;
  min-width: 0;
  cursor: pointer;
}
.row-top {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}
.row-top strong {
  font-size: 16px;
}
.room {
  font-size: 13px;
  color: var(--primary-dark);
  font-weight: 600;
}
.mark {
  font-size: 11px;
  padding: 1px 6px;
  border-radius: 999px;
}
.mark.fragile {
  background: rgba(245, 158, 11, 0.15);
  color: var(--warning);
}
.mark.liquid {
  background: rgba(59, 130, 246, 0.15);
  color: var(--info);
}
.row-sub {
  font-size: 12px;
  color: var(--text-secondary);
  margin-top: 3px;
}
.row-status {
  margin-top: 5px;
}
.badge {
  display: inline-block;
  font-size: 12px;
  padding: 2px 8px;
  border-radius: 999px;
}
.badge-none {
  background: var(--border);
  color: var(--text-secondary);
}
.badge-once {
  background: rgba(59, 130, 246, 0.15);
  color: var(--info);
}
.badge-multi {
  background: rgba(239, 68, 68, 0.15);
  color: var(--danger);
  font-weight: 700;
}
.row-print {
  flex-shrink: 0;
  padding: 8px 12px;
  font-size: 13px;
  box-shadow: none;
}

.bottom-bar {
  margin-top: 16px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.bottom-bar .btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
.printing-hint {
  text-align: center;
  font-size: 13px;
  color: var(--danger);
}

.print-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 12px;
  padding: 16px;
}
.print-label {
  border: 1px dashed #999;
  padding: 16px;
  text-align: center;
  page-break-inside: avoid;
  position: relative;
}
.print-code {
  font-size: 32px;
  font-weight: 800;
  margin-bottom: 8px;
}
.print-room {
  font-size: 20px;
  font-weight: 700;
  color: #d97706;
  margin-bottom: 8px;
}
.print-qr {
  width: 120px;
  height: 120px;
}
.print-tags {
  font-size: 12px;
  color: #666;
  margin-top: 6px;
}
.print-warn {
  font-size: 12px;
  color: #ef4444;
  margin-top: 4px;
}
.print-count {
  margin-top: 6px;
  font-size: 11px;
  color: #ef4444;
  font-weight: 700;
}
</style>
