<script setup lang="ts">
import { ref } from 'vue';
import { useRouter } from 'vue-router';
import { saveTask } from '../db';
import { uid, todayStr } from '../utils';
import type { MoveTask } from '../types';

const router = useRouter();
const title = ref('');
const from = ref('');
const to = ref('');
const date = ref(todayStr());
const roomsText = ref('卧室,客厅,厨房,卫生间');

async function submit() {
  if (!title.value || !from.value || !to.value) {
    alert('请填写完整信息');
    return;
  }
  const rooms = roomsText.value.split(/[,，]/).map((s) => s.trim()).filter(Boolean);
  const task: MoveTask = {
    id: uid(),
    title: title.value,
    from: from.value,
    to: to.value,
    date: date.value,
    rooms,
    boxes: [],
    createdAt: Date.now(),
  };
  await saveTask(task);
  router.push(`/task/${task.id}`);
}
</script>

<template>
  <div>
    <div class="header">
      <router-link to="/" class="back">←</router-link>
      <h1>新建搬家任务</h1>
    </div>
    <div class="page">
      <div class="card">
        <label class="label">任务名称</label>
        <input v-model="title" class="input" placeholder="例如：2024年9月搬家" />
      </div>
      <div class="card">
        <label class="label">旧住址</label>
        <input v-model="from" class="input" placeholder="出发地址" />
      </div>
      <div class="card">
        <label class="label">新住址</label>
        <input v-model="to" class="input" placeholder="目的地址" />
      </div>
      <div class="card">
        <label class="label">搬家日期</label>
        <input v-model="date" type="date" class="input" />
      </div>
      <div class="card">
        <label class="label">房间清单（用逗号分隔）</label>
        <input v-model="roomsText" class="input" />
      </div>
      <button class="btn btn-block" @click="submit">创建任务</button>
    </div>
  </div>
</template>
