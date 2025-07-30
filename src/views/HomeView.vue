<template>
  <el-container class="container">
    <el-header class="header">
      <span/>
      <div class="header-buttons">
        <el-button @click="openDataManager">数据管理</el-button>
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

    <!-- 标签管理对话框 -->
    <el-dialog v-model="tagManagerVisible" title="标签管理" width="600px">
      <div class="tag-manager">
        <div class="tag-manager-section">
          <h4>添加新标签</h4>
          <div class="add-tag-form">
            <el-input 
              v-model="newTagName" 
              placeholder="输入标签名称"
              style="width: 200px; margin-right: 10px;"
            />
            <el-select v-model="newTagType" placeholder="选择标签类型" style="width: 120px; margin-right: 10px;">
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
          <div class="tag-list">
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
      <el-dialog v-model="editTagVisible" title="编辑标签" width="400px" append-to-body>
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
  tags: [] as string[],
});

// 标签管理相关
const newTagName = ref('');
const newTagType = ref('primary');
const editingTag = reactive<Tag>({ name: '', type: 'primary' });
const originalTagName = ref('');

// 标签数据
const tags = ref<Tag[]>([
  { name: '工作', type: 'primary' },
  { name: '学习', type: 'success' },
  { name: '生活', type: 'info' },
  { name: '娱乐', type: 'warning' },
  { name: '健康', type: 'success' },
  { name: '旅行', type: 'warning' },
  { name: '生日', type: 'danger' },
  { name: '纪念日', type: 'danger' },
  { name: '考试', type: 'warning' },
  { name: '会议', type: 'primary' }
]);

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
  ElMessageBox.confirm(
    `确定要删除标签"${tagName}"吗？删除后，所有使用该标签的事件将不再显示该标签。`,
    '删除标签',
    {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning',
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

onMounted(() => {
  loadTags();
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
  eventForm.tags = [];
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
</style>
