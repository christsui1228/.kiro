---
inclusion: fileMatch
fileMatchPattern: 'OneManage_web/**/*'
---

# 前端代码风格规范

## 组合式 API 规范

### script setup 语法
- **必须使用** `<script setup>` 语法
- 按以下顺序组织代码：
  1. import 语句
  2. 常量定义
  3. 响应式数据（ref/reactive）
  4. computed 计算属性
  5. 函数定义
  6. 生命周期钩子

```vue
<script setup>
import { ref, computed, onMounted } from 'vue';
import { useMessage } from 'naive-ui';

// 常量
const POLLING_INTERVAL = 2000;
const MAX_RETRY_COUNT = 3;

// 响应式数据
const taskList = ref([]);
const isLoading = ref(false);

// 计算属性
const completedTasks = computed(() => 
  taskList.value.filter(t => t.status === 'completed')
);

// 函数定义
const loadTasks = async () => {
  // ...
};

// 生命周期
onMounted(() => {
  loadTasks();
});
</script>
```

### 响应式数据命名
- 使用语义化的名称，避免 `allXxx`、`dataXxx` 等模糊命名
- 布尔值使用 `is`、`has`、`should` 前缀
- 列表数据使用 `xxxList` 或复数形式

```javascript
// ✅ 推荐
const taskList = ref([]);
const isLoading = ref(false);
const hasError = ref(false);
const selectedTask = ref(null);

// ❌ 避免
const allTasks = ref([]);
const loading = ref(false);
const error = ref(false);
const task = ref(null);
```

## 函数规范

### 函数命名
- 使用 camelCase
- 动词开头，表达清晰的动作
- 事件处理函数使用 `handle` 前缀

```javascript
// ✅ 推荐
const loadTaskHistory = async () => {};
const handleUploadFinished = () => {};
const formatDate = (date) => {};

// ❌ 避免
const get = async () => {};
const upload = () => {};
const format = (date) => {};
```

### JSDoc 注释
- 所有导出的函数必须添加 JSDoc 注释
- 复杂的内部函数建议添加注释
- 注释使用简体中文

```javascript
/**
 * 加载任务历史记录
 * @param {number} limit - 返回的最大记录数
 * @returns {Promise<Array>} 任务列表
 */
const loadTaskHistory = async (limit = 50) => {
  // ...
};

/**
 * 格式化日期时间为本地化字符串
 * @param {string} dateStr - ISO 格式的日期字符串
 * @returns {string} 格式化后的日期字符串
 */
const formatDate = (dateStr) => {
  if (!dateStr) return '';
  const date = new Date(dateStr);
  return date.toLocaleString('zh-CN');
};
```

### 异步函数
- 使用 `async/await` 而非 Promise 链
- 必须处理错误情况
- 使用 `try-catch` 统一错误处理

```javascript
// ✅ 推荐
const loadData = async () => {
  isLoading.value = true;
  try {
    const data = await http.get('/api/data');
    dataList.value = data;
  } catch (error) {
    message.error('加载数据失败');
    console.error('加载数据失败:', error);
  } finally {
    isLoading.value = false;
  }
};

// ❌ 避免
const loadData = () => {
  http.get('/api/data')
    .then(data => {
      dataList.value = data;
    })
    .catch(error => {
      console.log(error);
    });
};
```

## 常量定义

### 魔法数字/字符串
- 提取为具名常量
- 使用 UPPER_SNAKE_CASE 命名
- 在文件顶部或函数外部定义

```javascript
// ✅ 推荐
const POLLING_INTERVAL = 2000;
const MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB
const API_TIMEOUT = 30000;

const startPolling = () => {
  setInterval(checkStatus, POLLING_INTERVAL);
};

// ❌ 避免
const startPolling = () => {
  setInterval(checkStatus, 2000);
};
```

### 枚举对象
- 使用对象定义枚举值
- 常量名使用 UPPER_SNAKE_CASE

```javascript
const TASK_STATUS = {
  PENDING: 'pending',
  PROCESSING: 'processing',
  COMPLETED: 'completed',
  FAILED: 'failed'
};

const STATUS_TEXT_MAP = {
  [TASK_STATUS.PENDING]: '等待中',
  [TASK_STATUS.PROCESSING]: '处理中',
  [TASK_STATUS.COMPLETED]: '已完成',
  [TASK_STATUS.FAILED]: '失败'
};
```

## 错误处理

### 统一错误处理模式
- API 调用必须使用 try-catch
- 根据 HTTP 状态码提供不同的错误提示
- 记录错误日志到控制台

```javascript
const loadData = async () => {
  try {
    const data = await http.get('/api/data');
    return data;
  } catch (error) {
    const status = error?.response?.status;
    
    if (status === 401) {
      message.error('未登录或登录已过期');
    } else if (status === 403) {
      message.error('没有权限访问');
    } else if (status === 404) {
      message.error('数据不存在');
    } else {
      message.error('加载数据失败');
    }
    
    console.error('加载数据失败:', error);
    throw error; // 如果需要上层处理，重新抛出
  }
};
```

### 用户提示
- 使用 Naive UI 的 message 组件
- 成功：`message.success()`
- 错误：`message.error()`
- 警告：`message.warning()`
- 信息：`message.info()`

## 组件规范

### 组件拆分原则
- 单个组件不超过 300 行（包括 template、script、style）
- 可复用的 UI 片段提取为独立组件
- 业务逻辑复杂时拆分为多个子组件

```
// 大型组件拆分示例
DataImport.vue (主组件)
├── UploadSection.vue (上传区域)
├── TaskList.vue (任务列表)
└── TaskItem.vue (单个任务项)
```

### Props 定义
- 使用 `defineProps` 定义
- 添加类型和默认值
- 添加注释说明

```javascript
const props = defineProps({
  // 任务 ID
  taskId: {
    type: String,
    required: true
  },
  // 是否显示详情
  showDetail: {
    type: Boolean,
    default: false
  },
  // 最大显示数量
  maxCount: {
    type: Number,
    default: 10
  }
});
```

### Emits 定义
- 使用 `defineEmits` 定义
- 事件名使用 kebab-case
- 添加注释说明

```javascript
const emit = defineEmits([
  'update:modelValue', // 更新 v-model 值
  'task-completed',    // 任务完成时触发
  'error'              // 发生错误时触发
]);

// 触发事件
emit('task-completed', taskData);
```

## 样式规范

### Scoped 样式
- 必须使用 `<style scoped>`
- 一个组件只有一个 style 块
- 避免深度选择器，优先通过 props 传递样式

```vue
<style scoped>
.container {
  padding: 24px;
}

.title {
  font-size: 18px;
  font-weight: 600;
}

/* 必要时使用深度选择器 */
.container :deep(.n-button) {
  margin-top: 16px;
}
</style>
```

### 类名命名
- 使用 kebab-case
- 语义化命名
- 避免缩写

```css
/* ✅ 推荐 */
.task-item {}
.upload-section {}
.status-badge {}

/* ❌ 避免 */
.ti {}
.us {}
.sb {}
```

## 导入规范

### 导入顺序
1. Vue 核心库
2. 第三方库
3. UI 组件库
4. 本地组件
5. API/工具函数
6. 类型定义

```javascript
// 1. Vue 核心
import { ref, computed, onMounted } from 'vue';
import { useRouter } from 'vue-router';

// 2. 第三方库
import axios from 'axios';

// 3. UI 组件库
import { NButton, NCard, useMessage } from 'naive-ui';

// 4. 本地组件
import TaskItem from './TaskItem.vue';

// 5. API/工具
import { taskApi } from '@/api/task';
import { formatDate } from '@/utils/format';

// 6. 类型（如果使用 TypeScript）
import type { Task } from '@/types';
```

### 路径别名
- 使用 `@/` 表示 `src/` 目录
- 相对路径仅用于同级或子级文件

```javascript
// ✅ 推荐
import { taskApi } from '@/api/task';
import TaskItem from './TaskItem.vue';

// ❌ 避免
import { taskApi } from '../../../api/task';
```

## 注释规范

### 代码注释
- 所有注释使用简体中文
- 复杂逻辑必须添加注释
- 注释说明"为什么"而非"是什么"

```javascript
// ✅ 推荐
// 使用 Map 存储轮询定时器，避免重复轮询同一任务
const pollingIntervals = new Map();

// 必须先停止旧的轮询，防止内存泄漏
if (pollingIntervals.has(taskId)) {
  clearInterval(pollingIntervals.get(taskId));
}

// ❌ 避免
// 创建 Map
const pollingIntervals = new Map();

// 清除定时器
clearInterval(pollingIntervals.get(taskId));
```

### TODO 注释
- 使用 `// TODO:` 标记待办事项
- 说明具体要做什么和原因

```javascript
// TODO: 添加任务取消功能，允许用户中止长时间运行的任务
// TODO: 优化轮询逻辑，使用 WebSocket 替代定时轮询
```

## 性能优化

### 避免不必要的响应式
- 不需要响应式的数据使用普通变量
- 大型静态数据使用 `shallowRef`

```javascript
// ✅ 推荐
const STATIC_CONFIG = { /* 大量配置 */ }; // 普通常量
const largeData = shallowRef([]); // 浅层响应式

// ❌ 避免
const STATIC_CONFIG = ref({ /* 大量配置 */ });
```

### 计算属性缓存
- 使用 computed 而非方法进行数据转换
- 避免在 computed 中执行副作用

```javascript
// ✅ 推荐
const completedCount = computed(() => 
  taskList.value.filter(t => t.status === 'completed').length
);

// ❌ 避免
const getCompletedCount = () => 
  taskList.value.filter(t => t.status === 'completed').length;
```

### 事件监听清理
- 在 `onBeforeUnmount` 中清理定时器、事件监听
- 避免内存泄漏

```javascript
const intervalId = ref(null);

onMounted(() => {
  intervalId.value = setInterval(checkStatus, 1000);
});

onBeforeUnmount(() => {
  if (intervalId.value) {
    clearInterval(intervalId.value);
  }
});
```

## API 调用规范

### 统一 API 封装
- 所有 API 调用必须在 `src/api/` 目录下封装
- 按业务模块划分文件
- 使用对象导出相关接口

```javascript
// src/api/task.js
import http from '@/utils/http';

export const taskApi = {
  /**
   * 获取任务列表
   * @param {Object} params - 查询参数
   * @returns {Promise<Array>}
   */
  async getList(params) {
    return await http.get('/api/tasks', { params });
  },

  /**
   * 获取任务详情
   * @param {string} taskId - 任务 ID
   * @returns {Promise<Object>}
   */
  async getDetail(taskId) {
    return await http.get(`/api/tasks/${taskId}`);
  },

  /**
   * 创建任务
   * @param {Object} data - 任务数据
   * @returns {Promise<Object>}
   */
  async create(data) {
    return await http.post('/api/tasks', data);
  }
};
```

### 组件中调用 API
- 在组件中导入 API 模块
- 使用 async/await 处理异步
- 统一错误处理

```javascript
import { taskApi } from '@/api/task';
import { useMessage } from 'naive-ui';

const message = useMessage();
const taskList = ref([]);
const isLoading = ref(false);

const loadTasks = async () => {
  isLoading.value = true;
  try {
    taskList.value = await taskApi.getList({ limit: 50 });
  } catch (error) {
    message.error('加载任务列表失败');
    console.error('加载任务失败:', error);
  } finally {
    isLoading.value = false;
  }
};
```

## 状态管理规范

### Pinia Store 定义
- 使用 Composition API 风格
- 按业务模块划分 store
- 导出 store 函数

```javascript
// src/stores/user.js
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';

export const useUserStore = defineStore('user', () => {
  // 状态
  const userInfo = ref(null);
  const token = ref(localStorage.getItem('token') || '');

  // 计算属性
  const isLoggedIn = computed(() => !!token.value);
  const userName = computed(() => userInfo.value?.name || '');

  // 方法
  const setToken = (newToken) => {
    token.value = newToken;
    localStorage.setItem('token', newToken);
  };

  const clearAuth = () => {
    token.value = '';
    userInfo.value = null;
    localStorage.removeItem('token');
  };

  return {
    userInfo,
    token,
    isLoggedIn,
    userName,
    setToken,
    clearAuth
  };
});
```

### 组件中使用 Store
```javascript
import { useUserStore } from '@/stores/user';

const userStore = useUserStore();

// 访问状态
console.log(userStore.userName);

// 调用方法
userStore.setToken('new-token');
```

## 表单处理规范

### 表单数据定义
- 使用 reactive 定义表单对象
- 使用 ref 定义表单引用

```javascript
import { reactive, ref } from 'vue';

const formRef = ref(null);
const formData = reactive({
  name: '',
  email: '',
  phone: ''
});

const formRules = {
  name: {
    required: true,
    message: '请输入姓名',
    trigger: 'blur'
  },
  email: {
    required: true,
    message: '请输入邮箱',
    trigger: 'blur'
  }
};
```

### 表单提交
```javascript
const handleSubmit = async () => {
  try {
    // 验证表单
    await formRef.value?.validate();
    
    // 提交数据
    await taskApi.create(formData);
    message.success('创建成功');
    
    // 重置表单
    Object.assign(formData, {
      name: '',
      email: '',
      phone: ''
    });
  } catch (error) {
    if (error?.errors) {
      // 表单验证失败
      return;
    }
    message.error('创建失败');
    console.error('创建失败:', error);
  }
};
```

## 列表渲染规范

### 使用 key
- v-for 必须添加 key
- key 使用唯一标识符（id）
- 避免使用 index 作为 key

```vue
<!-- ✅ 推荐 -->
<div v-for="task in taskList" :key="task.id">
  {{ task.name }}
</div>

<!-- ❌ 避免 -->
<div v-for="(task, index) in taskList" :key="index">
  {{ task.name }}
</div>
```

### 空状态处理
```vue
<template>
  <div v-if="isLoading">加载中...</div>
  <div v-else-if="taskList.length === 0">暂无数据</div>
  <div v-else>
    <div v-for="task in taskList" :key="task.id">
      {{ task.name }}
    </div>
  </div>
</template>
```

## 条件渲染规范

### v-if vs v-show
- 频繁切换使用 v-show
- 条件很少改变使用 v-if
- 初始不渲染使用 v-if

```vue
<!-- 频繁切换 -->
<div v-show="isVisible">内容</div>

<!-- 权限控制 -->
<div v-if="hasPermission">敏感内容</div>
```

## 事件处理规范

### 事件命名
- 使用 handle 前缀
- 语义化命名

```javascript
const handleClick = () => {};
const handleUploadSuccess = () => {};
const handleFormSubmit = () => {};
```

### 事件传参
```vue
<template>
  <!-- 不传参 -->
  <n-button @click="handleClick">点击</n-button>
  
  <!-- 传参 -->
  <n-button @click="() => handleDelete(task.id)">删除</n-button>
  
  <!-- 传递事件对象 -->
  <input @input="handleInput" />
</template>
```

## 路由使用规范

### 编程式导航
```javascript
import { useRouter } from 'vue-router';

const router = useRouter();

// 跳转
router.push('/tasks');
router.push({ name: 'TaskDetail', params: { id: '123' } });

// 替换
router.replace('/login');

// 返回
router.back();
```

### 路由参数获取
```javascript
import { useRoute } from 'vue-router';

const route = useRoute();

// 获取参数
const taskId = route.params.id;
const keyword = route.query.keyword;
```

## 工具函数规范

### 日期格式化
```javascript
// src/utils/format.js

/**
 * 格式化日期时间
 * @param {string|Date} date - 日期
 * @param {string} format - 格式，默认 'YYYY-MM-DD HH:mm:ss'
 * @returns {string}
 */
export const formatDate = (date, format = 'YYYY-MM-DD HH:mm:ss') => {
  if (!date) return '';
  const d = new Date(date);
  if (isNaN(d.getTime())) return '';
  
  const year = d.getFullYear();
  const month = String(d.getMonth() + 1).padStart(2, '0');
  const day = String(d.getDate()).padStart(2, '0');
  const hour = String(d.getHours()).padStart(2, '0');
  const minute = String(d.getMinutes()).padStart(2, '0');
  const second = String(d.getSeconds()).padStart(2, '0');
  
  return format
    .replace('YYYY', year)
    .replace('MM', month)
    .replace('DD', day)
    .replace('HH', hour)
    .replace('mm', minute)
    .replace('ss', second);
};
```

### 文件大小格式化
```javascript
/**
 * 格式化文件大小
 * @param {number} bytes - 字节数
 * @returns {string}
 */
export const formatFileSize = (bytes) => {
  if (bytes === 0) return '0 B';
  const k = 1024;
  const sizes = ['B', 'KB', 'MB', 'GB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return (bytes / Math.pow(k, i)).toFixed(2) + ' ' + sizes[i];
};
```

## 代码检查清单

提交代码前检查：

- [ ] 所有注释使用简体中文
- [ ] 函数添加了 JSDoc 注释
- [ ] 没有魔法数字/字符串
- [ ] 异步函数使用 try-catch
- [ ] 错误有适当的用户提示
- [ ] 响应式数据命名语义化
- [ ] 组件不超过 300 行
- [ ] 只有一个 `<style scoped>` 块
- [ ] 清理了 console.log 调试代码
- [ ] 清理了定时器和事件监听
- [ ] API 调用已封装到 api 目录
- [ ] v-for 添加了唯一 key
- [ ] 表单添加了验证规则
- [ ] 处理了空状态和加载状态
