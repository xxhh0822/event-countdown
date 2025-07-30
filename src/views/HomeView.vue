<template>
  <el-container class="container">
    <el-header class="header">
      <span/>
      <el-button type="primary" @click="openAddDialog">添加新事件</el-button>
    </el-header>

    <el-main>

      <div v-if="sortedEvents.length === 0" class="no-events">
        <p>暂无事件，快去添加一个吧！</p>
      </div>

      <el-row :gutter="20" v-else>
        <el-col :span="24" v-for="event in sortedEvents" :key="event.id">
          <el-card class="box-card">
            <template #header>
              <div class="card-header">
                <span>{{ event.name }}</span>
                <div>
                  <el-button class="button" text @click="editEvent(event)">编辑</el-button>
                  <el-button class="button" text type="danger" @click="deleteEvent(event.id)">删除</el-button>
                </div>
              </div>
            </template>
            <div class="countdown">
              <div class="time-block">{{ countdowns[event.id]?.days || '00' }}<span>天</span></div>
              <div class="time-block">{{ countdowns[event.id]?.hours || '00' }}<span>时</span></div>
              <div class="time-block">{{ countdowns[event.id]?.minutes || '00' }}<span>分</span></div>
              <div class="time-block">{{ countdowns[event.id]?.seconds || '00' }}<span>秒</span></div>
            </div>
          </el-card>
        </el-col>
      </el-row>
    </el-main>

    <!-- 添加/编辑事件对话框 -->
    <el-dialog v-model="dialogVisible" :title="isEditing ? '编辑事件' : '添加新事件'" width="90%">
      <el-form :model="eventForm" label-width="80px">
        <el-form-item label="事件名称">
          <el-input v-model="eventForm.name"></el-input>
        </el-form-item>
        <el-form-item label="设置时长">
          <div class="scroll-picker-container">
            <div class="scroll-picker-column">
              <ul ref="daysPicker" @scroll="handleScroll('days', $event)">
                <li v-for="n in 366" :key="n">{{ n - 1 }}</li>
              </ul>
              <div class="highlight"></div>
              <span>天</span>
            </div>
            <div class="scroll-picker-column">
              <ul ref="hoursPicker" @scroll="handleScroll('hours', $event)">
                <li v-for="n in 25" :key="n">{{ n - 1 }}</li>
              </ul>
              <div class="highlight"></div>
              <span>时</span>
            </div>
            <div class="scroll-picker-column">
              <ul ref="minutesPicker" @scroll="handleScroll('minutes', $event)">
                <li v-for="n in 61" :key="n">{{ n - 1 }}</li>
              </ul>
              <div class="highlight"></div>
              <span>分</span>
            </div>
          </div>
        </el-form-item>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="dialogVisible = false">取消</el-button>
          <el-button type="primary" @click="saveEvent">保存</el-button>
        </span>
      </template>
    </el-dialog>
  </el-container>
</template>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, computed, nextTick } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';
import dayjs from 'dayjs';
import duration from 'dayjs/plugin/duration';

dayjs.extend(duration);

interface CountdownEvent {
  id: number;
  name: string;
  date: string;
  creationDate: string;
}

interface Countdown {
  days: string;
  hours: string;
  minutes: string;
  seconds:string;
}

const events = ref<CountdownEvent[]>([]);
const countdowns = reactive<Record<number, Countdown>>({});
const dialogVisible = ref(false);
const daysPicker = ref<HTMLElement | null>(null);
const hoursPicker = ref<HTMLElement | null>(null);
const minutesPicker = ref<HTMLElement | null>(null);
const isEditing = ref(false);
const sortKey = ref<'remainingTime' | 'creationDate'>('remainingTime');

const eventForm = reactive({
  id: null as number | null,
  name: '',
  days: 0,
  hours: 0,
  minutes: 0,
});

// 数据持久化：使用 localStorage 模拟文件存储
const STORAGE_KEY = 'countdown-events';

const loadEvents = () => {
  const data = localStorage.getItem(STORAGE_KEY);
  if (data) {
    events.value = JSON.parse(data);
  }
};

const persistEvents = () => {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(events.value));
};

const calculateCountdown = (targetDate: string) => {
  const now = dayjs();
  const target = dayjs(targetDate);
  const diff = target.diff(now);

  if (diff <= 0) {
    return { days: '00', hours: '00', minutes: '00', seconds: '00' };
  }

  const duration = dayjs.duration(diff);
  return {
    days: String(Math.floor(duration.asDays())).padStart(2, '0'),
    hours: String(duration.hours()).padStart(2, '0'),
    minutes: String(duration.minutes()).padStart(2, '0'),
    seconds: String(duration.seconds()).padStart(2, '0'),
  };
};

const updateAllCountdowns = () => {
  events.value.forEach(event => {
    countdowns[event.id] = calculateCountdown(event.date);
  });
};

let timer: number;

onMounted(() => {
  loadEvents();
  updateAllCountdowns();
  timer = window.setInterval(updateAllCountdowns, 1000);
});

onUnmounted(() => {
  clearInterval(timer);
});

const sortedEvents = computed(() => {
  return [...events.value].sort((a, b) => {
    if (sortKey.value === 'remainingTime') {
      return dayjs(a.date).diff(dayjs()) - dayjs(b.date).diff(dayjs());
    } else {
      return dayjs(b.creationDate).diff(dayjs(a.creationDate));
    }
  });
});

const openAddDialog = () => {
  isEditing.value = false;
  eventForm.id = null;
  eventForm.name = '';
  eventForm.days = 0;
  eventForm.hours = 0;
  eventForm.minutes = 0;
  dialogVisible.value = true;
};

const handleScroll = (type: 'days' | 'hours' | 'minutes', event: Event) => {
  const target = event.target as HTMLElement;
  const scrollTop = target.scrollTop;
  const itemHeight = target.querySelector('li')?.offsetHeight || 40;
  const selectedIndex = Math.round(scrollTop / itemHeight);
  eventForm[type] = selectedIndex;
};

const editEvent = (event: CountdownEvent) => {
  isEditing.value = true;
  eventForm.id = event.id;
  eventForm.name = event.name;

  const now = dayjs();
  const target = dayjs(event.date);
  const diff = target.diff(now);

  if (diff > 0) {
    const duration = dayjs.duration(diff);
    eventForm.days = Math.floor(duration.asDays());
    eventForm.hours = duration.hours();
    eventForm.minutes = duration.minutes();
  } else {
    eventForm.days = 0;
    eventForm.hours = 0;
    eventForm.minutes = 0;
  }

  dialogVisible.value = true;

  nextTick(() => {
    const itemHeight = 40;
    if (daysPicker.value) daysPicker.value.scrollTop = eventForm.days * itemHeight;
    if (hoursPicker.value) hoursPicker.value.scrollTop = eventForm.hours * itemHeight;
    if (minutesPicker.value) minutesPicker.value.scrollTop = eventForm.minutes * itemHeight;
  });
};

const saveEvent = () => {
  if (!eventForm.name) {
    ElMessage.error('请填写事件名称');
    return;
  }
  if (eventForm.days === 0 && eventForm.hours === 0 && eventForm.minutes === 0) {
    ElMessage.error('请设置一个有效的时长');
    return;
  }

  const now = dayjs();
  const targetDate = now
    .add(eventForm.days, 'day')
    .add(eventForm.hours, 'hour')
    .add(eventForm.minutes, 'minute');

  if (isEditing.value && eventForm.id !== null) {
    // 编辑
    const index = events.value.findIndex(e => e.id === eventForm.id);
    if (index !== -1) {
      events.value[index] = { ...events.value[index], name: eventForm.name, date: targetDate.toISOString() };
    }
  } else {
    // 新增
    events.value.push({
      id: Date.now(),
      name: eventForm.name,
      date: targetDate.toISOString(),
      creationDate: now.toISOString(),
    });
  }
  
  persistEvents();
  updateAllCountdowns();
  dialogVisible.value = false;
  ElMessage.success('保存成功！');
};

const deleteEvent = (id: number) => {
  ElMessageBox.confirm('确定要删除这个事件吗？', '提示', {
    confirmButtonText: '确定',
    cancelButtonText: '取消',
    type: 'warning',
  })
    .then(() => {
      events.value = events.value.filter(e => e.id !== id);
      delete countdowns[id];
      persistEvents();
      ElMessage.success('删除成功！');
    })
    .catch(() => {
      // 取消删除
    });
};
</script>

<style scoped>
.scroll-picker-container {
  display: flex;
  justify-content: center;
  gap: 10px;
  width: 100%;
  padding: 0 20px;
}
.scroll-picker-column {
  position: relative;
  display: flex;
  align-items: center;
}
.scroll-picker-column ul {
  height: 200px; /* 5 items * 40px */
  width: 60px;
  overflow-y: scroll;
  list-style: none;
  padding: 80px 0; /* 2 items * 40px padding top/bottom */
  margin: 0;
  scroll-snap-type: y mandatory;
  -ms-overflow-style: none;  /* IE and Edge */
  scrollbar-width: none;  /* Firefox */
}
.scroll-picker-column ul::-webkit-scrollbar {
  display: none; /* Chrome, Safari, Opera */
}
.scroll-picker-column li {
  height: 40px;
  line-height: 40px;
  text-align: center;
  font-size: 20px;
  scroll-snap-align: center;
}
.scroll-picker-column .highlight {
  position: absolute;
  top: 50%;
  left: 0;
  transform: translateY(-50%);
  height: 40px;
  width: 60px;
  border-top: 1px solid #dcdfe6;
  border-bottom: 1px solid #dcdfe6;
  pointer-events: none; /* Allows scrolling through the highlight */
}
.scroll-picker-column span {
  margin-left: 8px;
  font-size: 16px;
}

.container {
  height: 100vh;
}
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #ebeef5;
}
.sort-options {
  margin-bottom: 20px;
}
.no-events {
  text-align: center;
  color: #909399;
  margin-top: 50px;
}
.box-card {
  margin-bottom: 20px;
  min-width: 300px;
}
.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.countdown {
  display: flex;
  justify-content: center;
  gap: 10px;
  font-size: 24px;
  font-weight: bold;
}
.time-block {
  display: flex;
  flex-direction: column;
  align-items: center;
  background-color: #f4f4f5;
  padding: 10px;
  border-radius: 5px;
}
.time-block span {
  font-size: 12px;
  font-weight: normal;
  color: #606266;
}
</style>
