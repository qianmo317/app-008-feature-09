<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { getTask, updateBox } from '../db';
import { parseQRContent, vibrateShort, playBeep, statusColor, statusLabel } from '../utils';
import type { MoveTask, Box } from '../types';

const route = useRoute();
const router = useRouter();
const task = ref<MoveTask | null>(null);
const foundBox = ref<Box | null>(null);
const manualCode = ref('');
const errorMsg = ref('');

let stream: MediaStream | null = null;
const videoRef = ref<HTMLVideoElement | null>(null);
const scanning = ref(false);

async function load() {
  task.value = await getTask(route.params.id as string);
}

const hasBarcodeDetector = 'BarcodeDetector' in window;

async function startCamera() {
  try {
    stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } });
    if (videoRef.value) {
      videoRef.value.srcObject = stream;
      await videoRef.value.play();
      scanning.value = true;
      if (hasBarcodeDetector) {
        scanLoop();
      } else {
        errorMsg.value = '当前浏览器不支持扫码，请使用手动输入';
      }
    }
  } catch (e) {
    errorMsg.value = '无法启动摄像头，请使用手动输入';
  }
}

async function scanLoop() {
  if (!scanning.value || !videoRef.value) return;
  const video = videoRef.value;
  if (video.readyState === video.HAVE_ENOUGH_DATA) {
    try {
      const detector = new (window as any).BarcodeDetector({ formats: ['qr_code'] });
      const barcodes = await detector.detect(video);
      if (barcodes.length > 0) {
        const text = barcodes[0].rawValue;
        handleScan(text);
        return;
      }
    } catch {
      // ignore
    }
  }
  requestAnimationFrame(scanLoop);
}

function handleScan(text: string) {
  const parsed = parseQRContent(text);
  if (!parsed.taskId || !parsed.code) {
    errorMsg.value = '无效二维码';
    return;
  }
  if (task.value && parsed.taskId !== task.value.id) {
    errorMsg.value = '该箱子不属于当前任务';
    return;
  }
  const b = task.value?.boxes.find((x) => x.code === parsed.code);
  if (!b) {
    errorMsg.value = '未找到箱子';
    return;
  }
  foundBox.value = b;
  vibrateShort();
  playBeep();
  errorMsg.value = '';
  scanning.value = false;
  if (stream) {
    stream.getTracks().forEach((t) => t.stop());
    stream = null;
  }
}

function searchManual() {
  if (!task.value || !manualCode.value) return;
  const b = task.value.boxes.find((x) => x.code.toLowerCase() === manualCode.value.trim().toLowerCase());
  if (b) {
    foundBox.value = b;
    errorMsg.value = '';
  } else {
    errorMsg.value = '未找到箱子';
  }
}

async function setStatus(status: Box['status']) {
  if (!foundBox.value || !task.value) return;
  foundBox.value.status = status;
  foundBox.value.updatedAt = Date.now();
  await updateBox(task.value.id, foundBox.value);
}

function reset() {
  foundBox.value = null;
  manualCode.value = '';
  errorMsg.value = '';
  startCamera();
}

onMounted(() => {
  load().then(startCamera);
});

onUnmounted(() => {
  scanning.value = false;
  if (stream) {
    stream.getTracks().forEach((t) => t.stop());
    stream = null;
  }
});
</script>

<template>
  <div v-if="task">
    <div class="header">
      <router-link :to="`/task/${task.id}`" class="back">←</router-link>
      <h1>扫码查箱</h1>
    </div>
    <div class="page">
      <div v-if="!foundBox">
        <video ref="videoRef" autoplay playsinline muted></video>
        <div style="margin-top:10px;">
          <input v-model="manualCode" class="input" placeholder="或手动输入箱号" @keyup.enter="searchManual" />
          <button class="btn btn-block" style="margin-top:8px;" @click="searchManual">查询</button>
        </div>
        <div v-if="errorMsg" style="color:var(--danger);margin-top:8px;font-size:14px;">{{ errorMsg }}</div>
      </div>

      <div v-else>
        <div class="card" style="text-align:center;">
          <div style="font-size:36px;font-weight:800;">{{ foundBox.code }}</div>
          <div style="font-size:16px;color:var(--text-secondary);margin-top:4px;">{{ foundBox.roomTo }}</div>
          <div style="margin-top:10px;">
            <span class="status-dot" :style="{background: statusColor(foundBox.status)}"></span>
            <span style="margin-left:6px;">{{ statusLabel(foundBox.status) }}</span>
          </div>
        </div>
        <div class="card">
          <div style="font-weight:700;margin-bottom:8px;">快速改状态</div>
          <div style="display:flex;flex-wrap:wrap;gap:8px;">
            <button class="tag" :class="{active: foundBox.status === 'packed'}" @click="setStatus('packed')">待打包</button>
            <button class="tag" :class="{active: foundBox.status === 'loaded'}" @click="setStatus('loaded')">已装车</button>
            <button class="tag" :class="{active: foundBox.status === 'arrived'}" @click="setStatus('arrived')">已到达</button>
            <button class="tag" :class="{active: foundBox.status === 'unpacked'}" @click="setStatus('unpacked')">已拆箱</button>
            <button class="tag" :class="{active: foundBox.status === 'damaged'}" @click="setStatus('damaged')">破损</button>
            <button class="tag" :class="{active: foundBox.status === 'missing'}" @click="setStatus('missing')">缺失</button>
          </div>
        </div>
        <div class="toolbar">
          <button class="btn" @click="router.push(`/task/${task.id}/box/${foundBox.code}`)">查看详情</button>
          <button class="btn btn-secondary" @click="reset">继续扫码</button>
        </div>
      </div>
    </div>
  </div>
</template>
