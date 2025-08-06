<template>
  <el-container class="container">
    <el-header class="header">
      <span/>
      <div class="header-buttons">
        <el-button @click="openTagManager">标签管理</el-button>
        <el-button type="primary" @click="openAddDialog">添加新事件</el-button>
      </div>
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
                <div class="event-info">
                  <span class="event-name">{{ event.name }}</span>
                  <div class="event-tags">
                    <el-tag 
                      v-for="tag in event.tags" 
                      :key="tag" 
                      :type="getTagType(tag)"
                      size="small"
                      class="event-tag"
                    >
                      {{ tag }}
                    </el-tag>
                  </div>
                </div>
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
        <el-form-item label="事件标签">
          <div class="tag-input-container">
            <el-select
              v-model="eventForm.tags"
              multiple
              filterable
              allow-create
              default-first-option
              placeholder="请选择或输入标签"
              style="width: 100%"
            >
              <el-option
                v-for="tag in availableTags"
                :key="tag"
                :label="tag"
                :value="tag"
              />
            </el-select>
          </div>
        </el-form-item>
        <el-form-item label="设置时长">
          <div class="time-input-container">
            <div class="time-input-group">
              <el-input
                v-model.number="eventForm.days"
                type="number"
                min="0"
                max="365"
                placeholder="天数"
                class="time-input"
                @input="onTimeInput('days')"
              />
              <span class="time-label">天</span>
            </div>
            <div class="time-input-group">
              <el-input
                v-model.number="eventForm.hours"
                type="number"
                min="0"
                max="24"
                placeholder="小时"
                class="time-input"
                @input="onTimeInput('hours')"
              />
              <span class="time-label">时</span>
            </div>
            <div class="time-input-group">
              <el-input
                v-model.number="eventForm.minutes"
                type="number"
                min="0"
                max="60"
                placeholder="分钟"
                class="time-input"
                @input="onTimeInput('minutes')"
              />
              <span class="time-label">分</span>
            </div>
          </div>
        </el-form-item>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="closeDialog">取消</el-button>
          <el-button type="primary" @click="saveEvent">保存</el-button>
        </span>
      </template>
    </el-dialog>

    <!-- 标签管理对话框 -->
    <el-dialog v-model="tagManagerVisible" title="标签管理" width="90%">
      <div class="tag-manager">
        <div class="tag-manager-section">
          <h4>添加新标签</h4>
          <div class="add-tag-form">
            <el-input 
              v-model="newTagName" 
              placeholder="输入标签名称"
            />
            <el-select v-model="newTagType" placeholder="选择标签类型">
              <el-option label="主要" value="primary" />
              <el-option label="成功" value="success" />
              <el-option label="信息" value="info" />
              <el-option label="警告" value="warning" />
              <el-option label="危险" value="danger" />
            </el-select>
            <el-button type="primary" @click="addNewTag" :disabled="!newTagName.trim()">添加</el-button>
          </div>
        </div>

        <div class="tag-manager-section">
          <h4>现有标签</h4>
          <div v-if="allTags.length === 0" class="no-tags">
            <p>暂无标签，请添加第一个标签</p>
          </div>
          <div v-else class="tag-list">
            <div v-for="tag in allTags" :key="tag.name" class="tag-item">
              <el-tag :type="tag.type" size="large">{{ tag.name }}</el-tag>
              <div class="tag-actions">
                <el-button size="small" @click="editTag(tag)">编辑</el-button>
                <el-button size="small" type="danger" @click="deleteTag(tag.name)">删除</el-button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- 编辑标签对话框 -->
      <el-dialog v-model="editTagVisible" title="编辑标签" width="90%" append-to-body>
        <el-form :model="editingTag" label-width="80px">
          <el-form-item label="标签名称">
            <el-input v-model="editingTag.name" />
          </el-form-item>
          <el-form-item label="标签类型">
            <el-select v-model="editingTag.type" placeholder="选择标签类型">
              <el-option label="主要" value="primary" />
              <el-option label="成功" value="success" />
              <el-option label="信息" value="info" />
              <el-option label="警告" value="warning" />
              <el-option label="危险" value="danger" />
            </el-select>
          </el-form-item>
        </el-form>
        <template #footer>
          <span class="dialog-footer">
            <el-button @click="editTagVisible = false">取消</el-button>
            <el-button type="primary" @click="saveTagEdit">保存</el-button>
          </span>
        </template>
      </el-dialog>
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
  tags: string[];
}

interface Countdown {
  days: string;
  hours: string;
  minutes: string;
  seconds:string;
}

interface Tag {
  name: string;
  type: string;
}

const events = ref<CountdownEvent[]>([]);
const countdowns = reactive<Record<number, Countdown>>({});
const dialogVisible = ref(false);
const tagManagerVisible = ref(false);
const editTagVisible = ref(false);

const isEditing = ref(false);
const sortKey = ref<'remainingTime' | 'creationDate'>('remainingTime');

const eventForm = reactive({
  id: null as number | null,
  name: '',
  days: 0,
  hours: 0,
  minutes: 0,
  tags: [] as string[],
});

// 标签管理相关
const newTagName = ref('');
const newTagType = ref('primary');
const editingTag = reactive<Tag>({ name: '', type: 'primary' });
const originalTagName = ref('');

// 标签数据
const tags = ref<Tag[]>([]);

// 获取所有可用标签名称
const availableTags = computed(() => {
  return tags.value.map(tag => tag.name);
});

// 获取所有标签（用于管理）
const allTags = computed(() => {
  return tags.value;
});

// 根据标签获取标签类型
const getTagType = (tagName: string) => {
  const tag = tags.value.find(t => t.name === tagName);
  return tag ? tag.type : 'info';
};

// 标签管理功能
const openTagManager = () => {
  tagManagerVisible.value = true;
};

const addNewTag = () => {
  const name = newTagName.value.trim();
  if (!name) {
    ElMessage.error('请输入标签名称');
    return;
  }
  
  if (tags.value.some(tag => tag.name === name)) {
    ElMessage.error('标签已存在');
    return;
  }
  
  tags.value.push({
    name: name,
    type: newTagType.value
  });
  
  saveTags();
  newTagName.value = '';
  newTagType.value = 'primary';
  ElMessage.success('标签添加成功！');
};

const editTag = (tag: Tag) => {
  editingTag.name = tag.name;
  editingTag.type = tag.type;
  originalTagName.value = tag.name;
  editTagVisible.value = true;
};

const saveTagEdit = () => {
  const name = editingTag.name.trim();
  if (!name) {
    ElMessage.error('请输入标签名称');
    return;
  }
  
  // 检查名称是否与其他标签冲突
  const existingTag = tags.value.find(tag => tag.name === name && tag.name !== originalTagName.value);
  if (existingTag) {
    ElMessage.error('标签名称已存在');
    return;
  }
  
  // 更新标签
  const tagIndex = tags.value.findIndex(tag => tag.name === originalTagName.value);
  if (tagIndex !== -1) {
    tags.value[tagIndex] = { ...editingTag };
    
    // 更新所有使用该标签的事件
    events.value.forEach(event => {
      const tagIndex = event.tags.indexOf(originalTagName.value);
      if (tagIndex !== -1) {
        event.tags[tagIndex] = name;
      }
    });
    
    saveTags();
    persistEvents();
    ElMessage.success('标签更新成功！');
    editTagVisible.value = false;
  }
};

const deleteTag = (tagName: string) => {
  // 检查是否有事件正在使用该标签
  const eventsUsingTag = events.value.filter(event => event.tags.includes(tagName));
  
  let message = `确定要删除标签"<strong>${tagName}</strong>"吗？`;
  
  if (eventsUsingTag.length > 0) {
    const eventNames = eventsUsingTag.map(event => event.name).join('、');
    message += `<br><br><span style="color: #e6a23c;">⚠️ 警告：以下事件正在使用此标签：</span><br><span style="color: #606266; font-size: 14px;">${eventNames}</span><br><br>删除后，这些事件将不再显示该标签。`;
  } else {
    message += '<br><br>删除后，所有使用该标签的事件将不再显示该标签。';
  }
  
  ElMessageBox.confirm(
    message,
    '删除标签',
    {
      confirmButtonText: '确定删除',
      cancelButtonText: '取消',
      type: eventsUsingTag.length > 0 ? 'warning' : 'info',
      dangerouslyUseHTMLString: true,
    }
  ).then(() => {
    // 从标签列表中删除
    tags.value = tags.value.filter(tag => tag.name !== tagName);
    
    // 从所有事件中移除该标签
    events.value.forEach(event => {
      event.tags = event.tags.filter(tag => tag !== tagName);
    });
    
    saveTags();
    persistEvents();
    ElMessage.success('标签删除成功！');
  }).catch(() => {
    // 取消删除
  });
};

// 标签数据持久化
const TAGS_STORAGE_KEY = 'countdown-tags';

const loadTags = () => {
  const data = localStorage.getItem(TAGS_STORAGE_KEY);
  if (data) {
    tags.value = JSON.parse(data);
  }
};

const saveTags = () => {
  localStorage.setItem(TAGS_STORAGE_KEY, JSON.stringify(tags.value));
};

// 数据持久化：使用 localStorage 模拟文件存储
const STORAGE_KEY = 'countdown-events';

const loadEvents = () => {
  const data = localStorage.getItem(STORAGE_KEY);
  if (data) {
    const parsedData = JSON.parse(data);
    // 确保旧数据兼容性
    events.value = parsedData.map((event: any) => ({
      ...event,
      tags: event.tags || []
    }));
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

// 动态设置容器高度
const setContainerHeight = () => {
  const vh = window.innerHeight * 0.01;
  document.documentElement.style.setProperty('--vh', `${vh}px`);
};

const closeDialog = () => {
  dialogVisible.value = false;
  setTimeout(() => {
    setContainerHeight();
    window.scrollTo(0, 0);
    const active = document.activeElement as HTMLElement | null;
    if (active && typeof active.blur === 'function') {
      active.blur();
    }
  }, 100);
};

onMounted(() => {
  loadTags();
  loadEvents();
  updateAllCountdowns();
  timer = window.setInterval(updateAllCountdowns, 1000);
  
  // 设置初始高度
  setContainerHeight();
  
  // 监听窗口大小变化
  window.addEventListener('resize', setContainerHeight);
  window.addEventListener('orientationchange', () => {
    setTimeout(setContainerHeight, 100);
  });
});

onUnmounted(() => {
  clearInterval(timer);
  window.removeEventListener('resize', setContainerHeight);
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
  eventForm.tags = [];
  dialogVisible.value = true;
};



const editEvent = (event: CountdownEvent) => {
  isEditing.value = true;
  eventForm.id = event.id;
  eventForm.name = event.name;
  eventForm.tags = [...event.tags];

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
};

const saveEvent = () => {
  // 修正输入范围
  if (eventForm.days < 0) eventForm.days = 0;
  if (eventForm.days > 365) eventForm.days = 365;
  if (eventForm.hours < 0) eventForm.hours = 0;
  if (eventForm.hours > 24) eventForm.hours = 24;
  if (eventForm.minutes < 0) eventForm.minutes = 0;
  if (eventForm.minutes > 60) eventForm.minutes = 60;

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
      events.value[index] = { 
        ...events.value[index], 
        name: eventForm.name, 
        date: targetDate.toISOString(),
        tags: eventForm.tags
      };
    }
  } else {
    // 新增
    events.value.push({
      id: Date.now(),
      name: eventForm.name,
      date: targetDate.toISOString(),
      creationDate: now.toISOString(),
      tags: eventForm.tags,
    });
  }
  
  persistEvents();
  updateAllCountdowns();
  closeDialog();
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

const onTimeInput = (type: 'days' | 'hours' | 'minutes') => {
  if (type === 'days') {
    if (eventForm.days < 0) eventForm.days = 0;
    if (eventForm.days > 365) eventForm.days = 365;
  } else if (type === 'hours') {
    if (eventForm.hours < 0) eventForm.hours = 0;
    if (eventForm.hours > 24) eventForm.hours = 24;
  } else if (type === 'minutes') {
    if (eventForm.minutes < 0) eventForm.minutes = 0;
    if (eventForm.minutes > 60) eventForm.minutes = 60;
  }
};
</script>

<style scoped>
.time-input-container {
  display: flex;
  justify-content: center;
  gap: 15px;
  width: 100%;
  padding: 0 20px;
}
.time-input-group {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}
.time-input {
  width: 120px;
}
.time-label {
  font-size: 14px;
  color: #606266;
  font-weight: 500;
}

.container {
  height: 100vh; /* fallback */
  height: calc(var(--vh, 1vh) * 100);
  min-height: calc(var(--vh, 1vh) * 100);
  max-height: calc(var(--vh, 1vh) * 100);
  overflow: hidden;
}
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #ebeef5;
}
.header-buttons {
  display: flex;
  gap: 10px;
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
  align-items: flex-start;
}
.event-info {
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.event-name {
  font-size: 16px;
  font-weight: bold;
}
.event-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
}
.event-tag {
  margin-right: 4px;
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
.tag-input-container {
  width: 100%;
}

/* 标签管理样式 */
.tag-manager {
  padding: 20px 0;
}
.tag-manager-section {
  margin-bottom: 30px;
}
.tag-manager-section h4 {
  margin-bottom: 15px;
  color: #303133;
}
.add-tag-form {
  display: flex;
  align-items: center;
  gap: 10px;
}
.add-tag-form .el-input {
  width: 200px;
}
.add-tag-form .el-select {
  width: 120px;
}
.tag-list {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
}
.tag-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px;
  border: 1px solid #ebeef5;
  border-radius: 6px;
  background-color: #fafafa;
}
.tag-actions {
  display: flex;
  gap: 5px;
}
.no-tags {
  text-align: center;
  color: #909399;
  padding: 20px;
  background-color: #fafafa;
  border-radius: 6px;
  border: 1px dashed #dcdfe6;
}

/* 移动端适配 */
@media (max-width: 768px) {
  .container {
    height: 100vh; /* fallback */
    height: calc(var(--vh, 1vh) * 100);
    min-height: calc(var(--vh, 1vh) * 100);
    max-height: calc(var(--vh, 1vh) * 100);
    overflow: hidden;
  }
  
  .header {
    padding: 10px 15px;
    min-height: 60px;
  }
  
  .header-buttons {
    gap: 8px;
  }
  
  .header-buttons .el-button {
    font-size: 14px;
    padding: 8px 12px;
  }
  
  .el-main {
    padding: 15px;
    overflow-y: auto;
    height: calc(calc(var(--vh, 1vh) * 100) - 60px);
  }
  
  .box-card {
    margin-bottom: 15px;
    min-width: auto;
  }
  
  .card-header {
    flex-direction: column;
    gap: 10px;
    align-items: flex-start;
  }
  
  .countdown {
    gap: 8px;
    font-size: 20px;
  }
  
  .time-block {
    padding: 8px;
    min-width: 50px;
  }
  
  .time-block span {
    font-size: 10px;
  }
  
  /* 对话框移动端适配 */
  .el-dialog {
    margin: 10px !important;
    width: calc(100% - 20px) !important;
    max-width: none !important;
  }
  
  .el-dialog__body {
    padding: 15px;
  }
  
  /* 时间输入框移动端适配 */
  .time-input-container {
    padding: 0 10px;
    gap: 10px;
  }
  
  .time-input-group {
    gap: 5px;
  }
  
  .time-input {
    width: 80px;
  }
  
  .time-label {
    font-size: 12px;
  }
  
  /* 标签管理移动端适配 */
  .tag-manager {
    padding: 15px 0;
  }
  
  .tag-manager-section {
    margin-bottom: 20px;
  }
  
  .tag-manager-section h4 {
    margin-bottom: 10px;
    font-size: 16px;
  }
  
  .add-tag-form {
    flex-direction: column;
    gap: 10px;
  }
  
  .add-tag-form .el-input,
  .add-tag-form .el-select {
    width: 100% !important;
    margin-right: 0 !important;
  }
  
  .add-tag-form .el-button {
    width: 100%;
    margin-top: 5px;
  }
  
  .tag-list {
    gap: 10px;
  }
  
  .tag-item {
    flex-direction: column;
    gap: 8px;
    padding: 8px;
    width: 100%;
  }
  
  .tag-actions {
    width: 100%;
    justify-content: center;
  }
  
  .tag-actions .el-button {
    flex: 1;
    max-width: 80px;
  }
}

/* 防止虚拟键盘弹出时页面缩放 */
@media screen and (max-width: 768px) {
  body {
    position: fixed;
    width: 100%;
    height: 100%;
    overflow: hidden;
  }
  
  #app {
    height: calc(var(--vh, 1vh) * 100);
    overflow: hidden;
  }
  
  .el-dialog__wrapper {
    position: fixed !important;
    top: 0 !important;
    left: 0 !important;
    right: 0 !important;
    bottom: 0 !important;
  }
  
  /* 对话框内容区域高度适配 */
  .el-dialog__body {
    max-height: calc(calc(var(--vh, 1vh) * 100) - 120px);
    overflow-y: auto;
  }
}
</style>
