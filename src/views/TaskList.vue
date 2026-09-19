<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import { getAllTasks, deleteTask } from '../db';
import type { MoveTask } from '../types';

const router = useRouter();
const tasks = ref<MoveTask[]>([]);

async function load() {
  tasks.value = await getAllTasks();
}

async function remove(id: string) {
  if (!confirm('确定删除该任务？')) return;
  await deleteTask(id);
  await load();
}

onMounted(load);
</script>

<template>
  <div>
    <div class="header">
      <h1>搬家打包追踪器</h1>
    </div>
    <div class="page">
      <button class="btn btn-block" @click="router.push('/new')">+ 新建搬家任务</button>
      <div v-if="tasks.length === 0" class="empty">暂无任务，点击上方按钮新建</div>
      <div v-for="t in tasks" :key="t.id" class="card" @click="router.push(`/task/${t.id}`)">
        <div style="display:flex;align-items:center;justify-content:space-between;gap:10px;">
          <div>
            <div style="font-weight:700;font-size:16px;">{{ t.title }}</div>
            <div style="font-size:13px;color:var(--text-secondary);margin-top:4px;">
              {{ t.from }} → {{ t.to }} · {{ t.date }} · {{ t.boxes.length }} 箱
            </div>
          </div>
          <button class="btn btn-secondary no-print" style="padding:8px 12px;font-size:13px;" @click.stop="remove(t.id)">删除</button>
        </div>
      </div>
    </div>
  </div>
</template>
