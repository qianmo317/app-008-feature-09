<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount, nextTick } from 'vue';
import { useRoute } from 'vue-router';
import { getTask, markBoxesPrinted } from '../db';
import { generateQRDataURL } from '../utils';
import type { MoveTask, Box } from '../types';

type TimeRange = 'all' | 'today' | '7d' | '30d' | 'custom';

const route = useRoute();
const task = ref<MoveTask | null>(null);

// 筛选条件
const keyword = ref('');
const roomFilter = ref('');
const timeRange = ref<TimeRange>('all');
const customDate = ref('');

// 勾选要打印的箱子 id
const selectedIds = ref<Set<string>>(new Set());
// 本次实际送去打印的箱子 id（只渲染这些进打印区）
const printIds = ref<string[]>([]);
// 打印中锁：同一张标签连点两次也只能出去一份
const printing = ref(false);

// QR 图按箱 id 缓存，避免每次打印重新生成
const qrCache = new Map<string, string>();

const timeOptions: { value: TimeRange; label: string }[] = [
  { value: 'all', label: '全部' },
  { value: 'today', label: '今天' },
  { value: '7d', label: '近 7 天' },
  { value: '30d', label: '近 30 天' },
  { value: 'custom', label: '指定日期' },
];

function localTodayStr(): string {
  const d = new Date();
  const p = (n: number) => String(n).padStart(2, '0');
  return `${d.getFullYear()}-${p(d.getMonth() + 1)}-${p(d.getDate())}`;
}

function pickTimeRange(v: TimeRange) {
  timeRange.value = v;
  if (v === 'custom' && !customDate.value) customDate.value = localTodayStr();
}

function currentRange(): { start: number; end: number } {
  switch (timeRange.value) {
    case 'today': {
      const d = new Date();
      d.setHours(0, 0, 0, 0);
      return { start: d.getTime(), end: Infinity };
    }
    case '7d':
      return { start: Date.now() - 7 * 86400000, end: Infinity };
    case '30d':
      return { start: Date.now() - 30 * 86400000, end: Infinity };
    case 'custom': {
      if (!customDate.value) return { start: 0, end: Infinity };
      const [y, m, d] = customDate.value.split('-').map(Number);
      const start = new Date(y, m - 1, d).getTime();
      return { start, end: start + 86400000 };
    }
    default:
      return { start: 0, end: Infinity };
  }
}

const filteredBoxes = computed<Box[]>(() => {
  const t = task.value;
  if (!t) return [];
  const kw = keyword.value.trim().toUpperCase();
  const range = currentRange();
  return t.boxes
    .filter(
      (b) =>
        (!roomFilter.value || b.roomTo === roomFilter.value) &&
        (!kw || b.code.toUpperCase().includes(kw)) &&
        b.createdAt >= range.start &&
        b.createdAt < range.end
    )
    .sort((a, b) => a.code.localeCompare(b.code));
});

const allFilteredSelected = computed(
  () => filteredBoxes.value.length > 0 && filteredBoxes.value.every((b) => selectedIds.value.has(b.id))
);

const selectedCount = computed(() => selectedIds.value.size);
const printedCount = computed(() => task.value?.boxes.filter((b) => (b.printCount ?? 0) > 0).length ?? 0);

function toggleOne(id: string) {
  const next = new Set(selectedIds.value);
  if (next.has(id)) next.delete(id);
  else next.add(id);
  selectedIds.value = next;
}

function toggleAllFiltered() {
  const next = new Set(selectedIds.value);
  if (allFilteredSelected.value) {
    for (const b of filteredBoxes.value) next.delete(b.id);
  } else {
    for (const b of filteredBoxes.value) next.add(b.id);
  }
  selectedIds.value = next;
}

function clearSelection() {
  selectedIds.value = new Set();
}

async function ensureQrs(t: MoveTask, ids: string[]) {
  await Promise.all(
    ids.map(async (id) => {
      if (qrCache.has(id)) return;
      const box = t.boxes.find((b) => b.id === id);
      if (!box) return;
      qrCache.set(id, await generateQRDataURL(t.id, box.code));
    })
  );
}

async function load() {
  const t = await getTask(route.params.id as string);
  task.value = t;
  if (!t) return;
  await ensureQrs(t, t.boxes.map((b) => b.id));
  // 默认全选，保持原来“一键全打”的顺手程度
  selectedIds.value = new Set(t.boxes.map((b) => b.id));
}

function waitForPrintImages(): Promise<void> {
  const root = document.getElementById('print-only');
  if (!root) return Promise.resolve();
  const imgs = Array.from(root.querySelectorAll('img'));
  const checks = imgs.map(
    (img) =>
      new Promise<void>((resolve) => {
        if ((img as HTMLImageElement).complete) {
          resolve();
          return;
        }
        img.addEventListener('load', () => resolve(), { once: true });
        img.addEventListener('error', () => resolve(), { once: true });
      })
  );
  return Promise.race([
    Promise.all(checks).then(() => undefined),
    new Promise<void>((resolve) => setTimeout(resolve, 1000)),
  ]);
}

// afterprint 立即解锁，事件不可用时 1 秒后兜底解锁（仍足以挡掉连点）
let unlockTimer = 0;
let unlockHandler: (() => void) | null = null;

function teardownUnlock() {
  if (unlockHandler) {
    window.removeEventListener('afterprint', unlockHandler);
    unlockHandler = null;
  }
  if (unlockTimer) {
    clearTimeout(unlockTimer);
    unlockTimer = 0;
  }
}

function armUnlock() {
  teardownUnlock();
  const finish = () => {
    teardownUnlock();
    printing.value = false;
    printIds.value = [];
  };
  unlockHandler = finish;
  window.addEventListener('afterprint', finish, { once: true });
  unlockTimer = window.setTimeout(finish, 1000);
}

async function doPrint(rawIds: string[]) {
  if (printing.value || !task.value) return;
  const ids = Array.from(new Set(rawIds));
  if (ids.length === 0) return;

  printing.value = true;
  try {
    // 打印前重新读取：打印期间被删掉的箱子不能再出现在清单里
    const fresh = await getTask(task.value.id);
    if (!fresh) {
      alert('任务不存在，无法打印');
      return;
    }
    const present = fresh.boxes.filter((b) => ids.includes(b.id));
    if (present.length === 0) {
      alert('要打印的箱子已被删除，无法打印');
      return;
    }
    if (present.length < ids.length) {
      alert(`有 ${ids.length - present.length} 个箱子已被删除，已自动跳过`);
    }

    task.value = fresh;
    selectedIds.value = new Set(
      [...selectedIds.value].filter((id) => fresh.boxes.some((b) => b.id === id))
    );

    await ensureQrs(fresh, present.map((b) => b.id));
    printIds.value = present.map((b) => b.id);
    await nextTick();
    await waitForPrintImages();

    window.print();

    // 记账：以仍存在的箱子为准，每个累加一次打印次数
    await markBoxesPrinted(fresh.id, printIds.value);
    task.value = await getTask(fresh.id);
  } catch {
    alert('打印失败，请重试');
    printIds.value = [];
  } finally {
    armUnlock();
  }
}

function printSelected() {
  void doPrint([...selectedIds.value]);
}

function printOne(id: string) {
  void doPrint([id]);
}

const printItems = computed(() =>
  printIds.value
    .map((id) => {
      const box = task.value?.boxes.find((b) => b.id === id);
      if (!box) return null;
      return { box, qr: qrCache.get(id) ?? '' };
    })
    .filter((x): x is { box: Box; qr: string } => x !== null)
);

function printBadge(count: number): { text: string; cls: string } {
  if (count <= 0) return { text: '未打印', cls: 'badge-none' };
  if (count === 1) return { text: '已打印 1 次', cls: 'badge-once' };
  return { text: `重打 ${count} 次`, cls: 'badge-multi' };
}

function formatTime(ts: number): string {
  const d = new Date(ts);
  const p = (n: number) => String(n).padStart(2, '0');
  const prefix = d.getFullYear() === new Date().getFullYear() ? '' : `${d.getFullYear()}年`;
  return `${prefix}${d.getMonth() + 1}月${d.getDate()}日 ${p(d.getHours())}:${p(d.getMinutes())}`;
}

onMounted(load);
onBeforeUnmount(teardownUnlock);
</script>

<template>
  <div v-if="task">
    <div class="header no-print">
      <router-link :to="`/task/${task.id}`" class="back">←</router-link>
      <h1>标签打印</h1>
    </div>

    <div class="page no-print">
      <!-- 筛选 -->
      <div class="card">
        <label class="label">按箱号查找</label>
        <input v-model="keyword" class="input" type="search" placeholder="如 A-014" />

        <label class="label" style="margin-top:12px;">按房间筛选</label>
        <select v-model="roomFilter" class="select">
          <option value="">全部房间</option>
          <option v-for="r in task.rooms" :key="r" :value="r">{{ r }}</option>
        </select>

        <label class="label" style="margin-top:12px;">按封箱时间筛选</label>
        <div style="display:flex;flex-wrap:wrap;gap:8px;">
          <span
            v-for="o in timeOptions"
            :key="o.value"
            class="tag"
            :class="{ active: timeRange === o.value }"
            @click="pickTimeRange(o.value)"
          >
            {{ o.label }}
          </span>
        </div>
        <input
          v-if="timeRange === 'custom'"
          v-model="customDate"
          class="input"
          type="date"
          style="margin-top:10px;"
        />
      </div>

      <!-- 批量选择 -->
      <div class="card select-bar">
        <label style="display:flex;align-items:center;gap:8px;cursor:pointer;font-weight:600;">
          <input
            type="checkbox"
            :checked="allFilteredSelected"
            :disabled="filteredBoxes.length === 0"
            @change="toggleAllFiltered"
          />
          全选当前结果
        </label>
        <span style="font-size:13px;color:var(--text-secondary);">
          共 {{ task.boxes.length }} 箱 · 已选 {{ selectedCount }} 张 · 已打印过 {{ printedCount }} 张
        </span>
        <button v-if="selectedCount > 0" class="link-btn" @click="clearSelection">清空选择</button>
      </div>

      <!-- 箱子清单 -->
      <div v-if="task.boxes.length === 0" class="empty">还没有箱子，先去封箱登记吧</div>
      <div v-else-if="filteredBoxes.length === 0" class="empty">没有符合筛选条件的箱子</div>
      <div v-else>
        <div
          v-for="b in filteredBoxes"
          :key="b.id"
          class="card box-row"
          :class="{ reprinted: (b.printCount ?? 0) >= 2 }"
          @click="toggleOne(b.id)"
        >
          <input
            type="checkbox"
            class="row-check"
            :checked="selectedIds.has(b.id)"
            @click.stop
            @change="toggleOne(b.id)"
          />
          <div class="row-main">
            <div class="row-line">
              <span class="box-code">{{ b.code }}</span>
              <span class="room">{{ b.roomTo }}</span>
              <span :class="['badge', printBadge(b.printCount ?? 0).cls]">
                {{ printBadge(b.printCount ?? 0).text }}
              </span>
            </div>
            <div class="row-sub">
              <span>{{ formatTime(b.createdAt) }} 封箱</span>
              <span v-if="b.lastPrintedAt">· 最近打印 {{ formatTime(b.lastPrintedAt) }}</span>
            </div>
          </div>
          <button
            class="mini-btn"
            :disabled="printing"
            @click.stop="printOne(b.id)"
          >
            单张打印
          </button>
        </div>
      </div>

      <!-- 底部批量打印 -->
      <div class="print-bar">
        <button class="btn btn-block" :disabled="selectedCount === 0 || printing" @click="printSelected">
          {{ printing ? '正在打印…' : `打印选中的 ${selectedCount} 张标签` }}
        </button>
      </div>
    </div>

    <!-- 实际打印区：只渲染本次选中的箱子 -->
    <div id="print-only" :class="{ single: printItems.length === 1 }">
      <div class="label-grid">
        <div v-for="it in printItems" :key="it.box.id" class="label-card">
          <div class="label-code">{{ it.box.code }}</div>
          <div class="label-room">{{ it.box.roomTo }}</div>
          <img :src="it.qr" class="label-qr" />
          <div class="label-tags">{{ it.box.tags.join(' · ') }}</div>
          <div class="label-warn">
            {{ it.box.fragile ? '易碎' : '' }} {{ it.box.liquid ? '液体禁运' : '' }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.select-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  flex-wrap: wrap;
}

.link-btn {
  border: none;
  background: none;
  color: var(--primary-dark);
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  padding: 0;
}

.box-row {
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: pointer;
}

.box-row.reprinted {
  border-color: var(--danger);
  background: color-mix(in srgb, var(--danger) 8%, var(--surface));
  box-shadow: 0 0 0 1px var(--danger) inset;
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
}

.row-line {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.box-code {
  font-size: 16px;
  font-weight: 800;
}

.room {
  font-size: 13px;
  font-weight: 700;
  color: var(--primary-dark);
}

.row-sub {
  margin-top: 4px;
  font-size: 12px;
  color: var(--text-secondary);
}

.badge {
  margin-left: auto;
  padding: 2px 8px;
  border-radius: 999px;
  font-size: 11px;
  font-weight: 700;
  white-space: nowrap;
}

.badge-none {
  background: var(--border);
  color: var(--text-secondary);
}

.badge-once {
  background: color-mix(in srgb, var(--info) 15%, transparent);
  color: var(--info);
}

.badge-multi {
  background: var(--danger);
  color: #fff;
}

.mini-btn {
  flex-shrink: 0;
  border: 1px solid var(--primary-dark);
  background: var(--bg);
  color: var(--primary-dark);
  border-radius: 999px;
  padding: 6px 12px;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
}

.mini-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.print-bar {
  position: sticky;
  bottom: 0;
  padding: 12px 0 4px;
  background: var(--bg);
}

.btn:disabled {
  opacity: 0.55;
  cursor: not-allowed;
}

/* 打印区屏幕上隐藏，仅在打印时显示（全局 @media print 负责可见性） */
#print-only {
  display: none;
}

@media print {
  #print-only {
    display: block;
  }

  .label-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 12px;
    padding: 16px;
  }

  #print-only.single .label-grid {
    grid-template-columns: 1fr;
    max-width: 360px;
    margin: 0 auto;
  }

  .label-card {
    border: 1px dashed #999;
    padding: 16px;
    text-align: center;
    page-break-inside: avoid;
  }

  .label-code {
    font-size: 32px;
    font-weight: 800;
    margin-bottom: 8px;
  }

  .label-room {
    font-size: 20px;
    font-weight: 700;
    color: #d97706;
    margin-bottom: 8px;
  }

  .label-qr {
    width: 120px;
    height: 120px;
  }

  .label-tags {
    font-size: 12px;
    color: #666;
    margin-top: 6px;
  }

  .label-warn {
    font-size: 12px;
    color: #ef4444;
    margin-top: 4px;
  }
}
</style>
