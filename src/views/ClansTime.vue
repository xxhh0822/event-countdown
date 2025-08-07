<template>
  <el-container style="height: 100vh; height: calc(var(--vh, 1vh) * 100);">
    <el-header height="56px">
      <el-tabs v-model="activeModule" type="card" stretch>
        <el-tab-pane
          v-for="(module, idx) in modules"
          :key="module.id"
          :label="module.name"
          :name="String(idx)"
        />
      </el-tabs>
    </el-header>
    <el-main>
      <el-card v-if="currentModule">
        <div style="font-weight:bold; font-size:16px; margin-bottom:12px;">{{ currentModule.name }}</div>
        <el-row :gutter="12">
          <el-col :span="24" v-for="timer in sortedTimers(currentModule.timers)" :key="timer.id">
            <el-card style="margin-bottom: 12px;">
              <div style="display:flex; align-items:center; justify-content:space-between;">
                <div style="display:flex; align-items:center; gap:16px;">
                  <span>{{ formatTimer(timer) }}</span>
                  <el-button
                    v-if="timer.remark"
                    type="primary"
                    size="small"
                    disabled
                    style="pointer-events: none; cursor: default;"
                  >{{ timer.remark }}</el-button>
                </div>
                <el-button size="small" @click="editTimer(timer.id)">编辑</el-button>
              </div>
            </el-card>
          </el-col>
        </el-row>
      </el-card>
    </el-main>
    <el-footer height="56px" style="position: relative; z-index: 1000; background: #fff; border-top: 1px solid #ebeef5;">
      <el-button type="primary" @click="openModuleManager" style="width: 100%;">模块管理</el-button>
    </el-footer>
    <!-- 模块管理弹窗 -->
    <el-dialog v-model="moduleManagerVisible" title="模块管理" width="90%">
      <el-form @submit.prevent>
        <el-form-item>
          <el-input v-model="newModuleName" placeholder="新模块名称" />
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="addModule" :disabled="!newModuleName.trim()" style="width:100%">添加模块</el-button>
        </el-form-item>
      </el-form>
      <el-divider>模块列表</el-divider>
      <div style="max-height: 400px; overflow-y: auto;">
        <el-row :gutter="16">
          <el-col :span="24" v-for="(module, idx) in modules" :key="module.id">
            <el-card shadow="always" style="margin-bottom: 16px;">
              <div style="display:flex; justify-content:space-between; align-items:center;">
                <span>{{ module.name }}</span>
                <el-button size="small" type="danger" @click="deleteModule(idx)">删除</el-button>
              </div>
            </el-card>
          </el-col>
        </el-row>
      </div>
    </el-dialog>
    <!-- 倒计时编辑弹窗 -->
    <el-dialog v-model="timerEditVisible" title="编辑倒计时" width="90%">
      <el-form :model="editTimerForm" label-width="60px">
        <el-form-item label="天">
          <el-input-number v-model="editTimerForm.days" :min="0" :max="365" style="width:100%" />
        </el-form-item>
        <el-form-item label="时">
          <el-input-number v-model="editTimerForm.hours" :min="0" :max="23" style="width:100%" />
        </el-form-item>
        <el-form-item label="分">
          <el-input-number v-model="editTimerForm.minutes" :min="0" :max="59" style="width:100%" />
        </el-form-item>
        <el-form-item label="备注">
          <el-input v-model="editTimerForm.remark" placeholder="请输入备注" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="timerEditVisible = false">取消</el-button>
        <el-button type="primary" @click="saveTimerEdit">保存</el-button>
      </template>
    </el-dialog>
  </el-container>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, watch, reactive } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';

interface Timer {
  id: number;
  days: number;
  hours: number;
  minutes: number;
  seconds: number;
  target: number;
  remark?: string;
}
interface ClanModule {
  id: number;
  name: string;
  timers: Timer[];
}
const STORAGE_KEY = 'clansTime-modules';
function loadModules(): ClanModule[] {
  const data = localStorage.getItem(STORAGE_KEY);
  if (data) {
    try {
      return JSON.parse(data);
    } catch {
      return [initModule('模块一')];
    }
  }
  return [initModule('模块一')];
}
const modules = ref<ClanModule[]>(loadModules());
const activeModule = ref('0');
const moduleManagerVisible = ref(false);
const newModuleName = ref('');
const timerEditVisible = ref(false);
const editTimerForm = reactive({ days: 1, hours: 0, minutes: 0, remark: '' });
let editTimerIdx = 0;
const currentModule = computed(() => modules.value[Number(activeModule.value)]);
function persistModules() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(modules.value));
}
watch(modules, persistModules, { deep: true });
function addModule() {
  modules.value.push(initModule(newModuleName.value.trim()));
  newModuleName.value = '';
  ElMessage.success('模块已添加');
  // 不关闭弹窗
}
function deleteModule(idx: number) {
  ElMessageBox.confirm(
    '确定要删除该模块吗？',
    '删除确认',
    {
      confirmButtonText: '删除',
      cancelButtonText: '取消',
      type: 'warning',
    }
  ).then(() => {
    modules.value.splice(idx, 1);
    ElMessage.success('模块已删除');
  }).catch(() => {
    // 用户取消，无操作
  });
}
function initModule(name: string): ClanModule {
  const now = Date.now();
  return {
    id: now + Math.floor(Math.random() * 10000),
    name,
    timers: Array.from({ length: 8 }, (_, i) => initTimer(now + i)),
  };
}
function initTimer(id: number): Timer {
  const now = Date.now();
  const days = 1, hours = 0, minutes = 0;
  return {
    id,
    days,
    hours,
    minutes,
    seconds: 0,
    target: now + days * 86400000 + hours * 3600000 + minutes * 60000,
    remark: ''
  };
}
// 倒计时逻辑
const currentTime = ref(Date.now());
let timerInterval: number;

// 移动端视口高度处理
const setContainerHeight = () => {
  const vh = window.innerHeight * 0.01;
  document.documentElement.style.setProperty('--vh', `${vh}px`);
};

onMounted(() => {
  // 设置初始高度
  setContainerHeight();
  
  // 监听窗口大小变化
  window.addEventListener('resize', setContainerHeight);
  window.addEventListener('orientationchange', () => {
    setTimeout(setContainerHeight, 100);
  });
  
  timerInterval = window.setInterval(() => {
    currentTime.value = Date.now();
  }, 1000);
});
function formatTimer(timer: Timer) {
  const remain = getRemain(timer);
  return `${remain.days}天${remain.hours}时${remain.minutes}分${remain.seconds}秒`;
}
function getRemain(timer: Timer) {
  let diff = timer.target - currentTime.value;
  if (diff < 0) diff = 0;
  const days = Math.floor(diff / 86400000);
  const hours = Math.floor((diff % 86400000) / 3600000);
  const minutes = Math.floor((diff % 3600000) / 60000);
  const seconds = Math.floor((diff % 60000) / 1000);
  return { days, hours, minutes, seconds };
}
function sortedTimers(timers: Timer[]) {
  return [...timers].sort((a, b) => (a.target - currentTime.value) - (b.target - currentTime.value));
}
function editTimer(timerId: number) {
  const timers = currentModule.value.timers;
  const idx = timers.findIndex(t => t.id === timerId);
  if (idx === -1) return;
  editTimerIdx = idx;
  const timer = timers[idx];
  const remain = getRemain(timer);
  if (remain.days === 0 && remain.hours === 0 && remain.minutes === 0 && remain.seconds === 0) {
    editTimerForm.days = 0;
    editTimerForm.hours = 0;
    editTimerForm.minutes = 0;
  } else {
    editTimerForm.days = remain.days;
    editTimerForm.hours = remain.hours;
    editTimerForm.minutes = remain.minutes;
  }
  editTimerForm.remark = timer.remark || '';
  timerEditVisible.value = true;
}
function saveTimerEdit() {
  const timer = currentModule.value.timers[editTimerIdx];
  timer.days = editTimerForm.days;
  timer.hours = editTimerForm.hours;
  timer.minutes = editTimerForm.minutes;
  timer.target = Date.now() + timer.days * 86400000 + timer.hours * 3600000 + timer.minutes * 60000;
  timer.remark = editTimerForm.remark;
  timerEditVisible.value = false;
  ElMessage.success('倒计时已更新');
}
function openModuleManager() {
  moduleManagerVisible.value = true;
}
</script> 