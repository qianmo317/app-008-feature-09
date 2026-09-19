<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { useRoute } from 'vue-router';
import { getTask } from '../db';
import { generateQRDataURL } from '../utils';
import type { MoveTask, Box } from '../types';

const route = useRoute();
const task = ref<MoveTask | null>(null);
const items = ref<{ box: Box; qr: string }[]>([]);



async function load() {
  const t = await getTask(route.params.id as string);
  task.value = t;
  if (!t) return;
  const list = await Promise.all(
    t.boxes.map(async (b) => ({
      box: b,
      qr: await generateQRDataURL(t.id, b.code),
    }))
  );
  items.value = list;
}

function printLabels() {
  window.print();
}

onMounted(load);
</script>

<template>
  <div v-if="task">
    <div class="header no-print">
      <router-link :to="`/task/${task.id}`" class="back">←</router-link>
      <h1>标签打印</h1>
      <button class="btn no-print" style="padding:8px 14px;font-size:14px;" @click="printLabels">打印</button>
    </div>
    <div class="page no-print">
      <div class="empty">点击下方按钮预览并打印 A4 标签页</div>
      <button class="btn btn-block" @click="printLabels">预览并打印</button>
    </div>

    <div id="print-only">
      <div style="display:grid;grid-template-columns:repeat(2, 1fr);gap:12px;padding:16px;">
        <div v-for="it in items" :key="it.box.id" style="border:1px dashed #999;padding:16px;text-align:center;page-break-inside:avoid;">
          <div style="font-size:32px;font-weight:800;margin-bottom:8px;">{{ it.box.code }}</div>
          <div style="font-size:20px;font-weight:700;color:#d97706;margin-bottom:8px;">{{ it.box.roomTo }}</div>
          <img :src="it.qr" style="width:120px;height:120px;" />
          <div style="font-size:12px;color:#666;margin-top:6px;">{{ it.box.tags.join(' · ') }}</div>
          <div style="font-size:12px;color:#ef4444;margin-top:4px;">
            {{ it.box.fragile ? '易碎' : '' }} {{ it.box.liquid ? '液体禁运' : '' }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
