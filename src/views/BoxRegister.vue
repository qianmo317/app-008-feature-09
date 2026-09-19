<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { getTask, saveTask } from '../db';
import { uid, compressImage, generateBoxCode } from '../utils';
import type { MoveTask, Box } from '../types';

const route = useRoute();
const router = useRouter();
const task = ref<MoveTask | null>(null);
const roomTo = ref('');
const tags = ref<string[]>([]);
const fragile = ref(false);
const liquid = ref(false);
const weightKg = ref<number | null>(null);
const note = ref('');
const photoData = ref<string>('');

const tagOptions = ['厨房', '衣物', '证件', '书籍', '电子', '杂物'];

async function load() {
  task.value = await getTask(route.params.id as string);
  if (task.value && task.value.rooms.length > 0) {
    roomTo.value = task.value.rooms[0];
  }
}

function toggleTag(t: string) {
  if (tags.value.includes(t)) tags.value = tags.value.filter((x) => x !== t);
  else tags.value.push(t);
}

async function onPhoto(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0];
  if (!file) return;
  photoData.value = await compressImage(file);
}

async function submit() {
  if (!task.value || !roomTo.value) {
    alert('请选择目标房间');
    return;
  }
  const code = generateBoxCode(task.value, roomTo.value);
  const box: Box = {
    id: uid(),
    code,
    roomFrom: '',
    roomTo: roomTo.value,
    tags: [...tags.value],
    fragile: fragile.value,
    liquid: liquid.value,
    photo: photoData.value || undefined,
    weightKg: weightKg.value ?? undefined,
    status: 'packed',
    note: note.value || undefined,
    createdAt: Date.now(),
    updatedAt: Date.now(),
  };
  task.value.boxes.push(box);
  await saveTask(task.value);
  if (confirm(`箱号 ${code} 已生成，是否继续封箱？`)) {
    tags.value = [];
    fragile.value = false;
    liquid.value = false;
    weightKg.value = null;
    note.value = '';
    photoData.value = '';
  } else {
    router.push(`/task/${task.value.id}`);
  }
}

onMounted(load);
</script>

<template>
  <div v-if="task">
    <div class="header">
      <router-link :to="`/task/${task.id}`" class="back">←</router-link>
      <h1>封箱登记</h1>
    </div>
    <div class="page">
      <div class="card">
        <label class="label">目标房间</label>
        <select v-model="roomTo" class="select">
          <option v-for="r in task.rooms" :key="r" :value="r">{{ r }}</option>
        </select>
      </div>
      <div class="card">
        <label class="label">物品标签</label>
        <div style="display:flex;flex-wrap:wrap;gap:8px;margin-top:6px;">
          <span v-for="t in tagOptions" :key="t" class="tag" :class="{active: tags.includes(t)}" @click="toggleTag(t)">{{ t }}</span>
        </div>
      </div>
      <div class="card">
        <label class="label">特殊标记</label>
        <div style="display:flex;gap:12px;margin-top:6px;">
          <label style="display:flex;align-items:center;gap:6px;cursor:pointer;">
            <input type="checkbox" v-model="fragile" /> 易碎
          </label>
          <label style="display:flex;align-items:center;gap:6px;cursor:pointer;">
            <input type="checkbox" v-model="liquid" /> 液体禁运
          </label>
        </div>
      </div>
      <div class="card">
        <label class="label">重量 (kg，可选)</label>
        <input v-model.number="weightKg" type="number" class="input" placeholder="例如：12.5" />
      </div>
      <div class="card">
        <label class="label">箱内照片</label>
        <input type="file" accept="image/*" capture="environment" @change="onPhoto" class="input" style="padding:8px;" />
        <img v-if="photoData" :src="photoData" style="width:100%;margin-top:10px;border-radius:10px;" />
      </div>
      <div class="card">
        <label class="label">备注</label>
        <textarea v-model="note" class="textarea" rows="2"></textarea>
      </div>
      <button class="btn btn-block" @click="submit">生成箱号并保存</button>
    </div>
  </div>
</template>
