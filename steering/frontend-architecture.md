---
inclusion: fileMatch
fileMatchPattern: 'OneManage_web/**/*'
---

# 前端架构规范

## 目录结构

```
OneManage_web/
├── public/                 # 静态资源
│   └── favicon.ico
├── src/
│   ├── api/               # API 接口封装
│   │   ├── task.js
│   │   ├── user.js
│   │   └── index.js
│   ├── assets/            # 资源文件
│   │   ├── images/
│   │   └── styles/
│   ├── components/        # 通用组件
│   │   ├── common/        # 基础组件
│   │   └── business/      # 业务组件
│   ├── composables/       # 组合式函数
│   │   ├── usePolling.js
│   │   └── useTable.js
│   ├── router/            # 路由配置
│   │   └── index.js
│   ├── stores/            # Pinia 状态管理
│   │   ├── user.js
│   │   └── app.js
│   ├── utils/             # 工具函数
│   │   ├── http.js
│   │   ├── format.js
│   │   └── validate.js
│   ├── views/             # 页面组件
│   │   ├── Home.vue
│   │   └── TaskList.vue
│   ├── App.vue            # 根组件
│   └── main.js            # 入口文件
├── .env                   # 环境变量
├── .env.development
├── .env.production
├── index.html
├── package.json
└── vite.config.js
```

## 组件设计原则

### 单一职责
- 每个组件只负责一个功能
- 复杂组件拆分为多个子组件
- 业务逻辑和 UI 展示分离

```
// ❌ 避免：一个组件做太多事
TaskManagement.vue (500+ 行)

// ✅ 推荐：拆分为多个组件
TaskManagement.vue (主容器)
├── TaskFilter.vue (筛选器)
├── TaskTable.vue (表格)
└── TaskDialog.vue (对话框)
```

### 组件分类

#### 1. 基础组件 (components/common/)
- 纯 UI 组件，无业务逻辑
- 高度可复用
- 通过 props 接收数据

```vue
<!-- components/common/StatusBadge.vue -->
<script setup>
const props = defineProps({
  status: {
    type: String,
    required: true
  }
});

const statusConfig = {
  success: { color: 'success', text: '成功' },
  error: { color: 'error', text: '失败' },
  pending: { color: 'warning', text: '处理中' }
};
</script>

<template>
  <n-tag :type="statusConfig[status].color">
    {{ statusConfig[status].text }}
  </n-tag>
</template>
```

#### 2. 业务组件 (components/business/)
- 包含特定业务逻辑
- 可能调用 API
- 可能使用 store

```vue
<!-- components/business/TaskCard.vue -->
<script setup>
import { taskApi } from '@/api/task';
import { useMessage } from 'naive-ui';

const props = defineProps({
  task: {
    type: Object,
    required: true
  }
});

const emit = defineEmits(['refresh']);
const message = useMessage();

const handleDelete = async () => {
  try {
    await taskApi.delete(props.task.id);
    message.success('删除成功');
    emit('refresh');
  } catch (error) {
    message.error('删除失败');
  }
};
</script>
```

#### 3. 页面组件 (views/)
- 路由对应的页面
- 组合多个组件
- 处理页面级状态

```vue
<!-- views/TaskList.vue -->
<script setup>
import { ref, onMounted } from 'vue';
import TaskFilter from '@/components/business/TaskFilter.vue';
import TaskTable from '@/components/business/TaskTable.vue';
import { taskApi } from '@/api/task';

const taskList = ref([]);
const filterParams = ref({});

const loadTasks = async () => {
  taskList.value = await taskApi.getList(filterParams.value);
};

const handleFilterChange = (params) => {
  filterParams.value = params;
  loadTasks();
};

onMounted(() => {
  loadTasks();
});
</script>

<template>
  <div class="task-list-page">
    <TaskFilter @change="handleFilterChange" />
    <TaskTable :data="taskList" @refresh="loadTasks" />
  </div>
</template>
```

## 组合式函数 (Composables)

### 定义
- 封装可复用的逻辑
- 使用 `use` 前缀命名
- 返回响应式数据和方法

```javascript
// composables/usePolling.js
import { ref, onBeforeUnmount } from 'vue';

/**
 * 轮询 Hook
 * @param {Function} callback - 轮询执行的函数
 * @param {number} interval - 轮询间隔（毫秒）
 * @returns {Object}
 */
export const usePolling = (callback, interval = 2000) => {
  const isPolling = ref(false);
  let timerId = null;

  const start = () => {
    if (isPolling.value) return;
    
    isPolling.value = true;
    timerId = setInterval(callback, interval);
  };

  const stop = () => {
    if (timerId) {
      clearInterval(timerId);
      timerId = null;
    }
    isPolling.value = false;
  };

  onBeforeUnmount(() => {
    stop();
  });

  return {
    isPolling,
    start,
    stop
  };
};
```

### 使用
```javascript
import { usePolling } from '@/composables/usePolling';

const checkTaskStatus = async () => {
  const status = await taskApi.getStatus(taskId);
  if (status === 'completed') {
    polling.stop();
  }
};

const polling = usePolling(checkTaskStatus, 2000);

// 开始轮询
polling.start();

// 停止轮询
polling.stop();
```

### 常用 Composables

#### useTable - 表格逻辑
```javascript
// composables/useTable.js
import { ref, reactive } from 'vue';

export const useTable = (fetchFn) => {
  const data = ref([]);
  const loading = ref(false);
  const pagination = reactive({
    page: 1,
    pageSize: 10,
    total: 0
  });

  const loadData = async () => {
    loading.value = true;
    try {
      const result = await fetchFn({
        page: pagination.page,
        pageSize: pagination.pageSize
      });
      data.value = result.items;
      pagination.total = result.total;
    } finally {
      loading.value = false;
    }
  };

  const handlePageChange = (page) => {
    pagination.page = page;
    loadData();
  };

  return {
    data,
    loading,
    pagination,
    loadData,
    handlePageChange
  };
};
```

#### useDialog - 对话框逻辑
```javascript
// composables/useDialog.js
import { ref } from 'vue';

export const useDialog = () => {
  const visible = ref(false);
  const data = ref(null);

  const open = (initialData = null) => {
    data.value = initialData;
    visible.value = true;
  };

  const close = () => {
    visible.value = false;
    data.value = null;
  };

  return {
    visible,
    data,
    open,
    close
  };
};
```

## 状态管理策略

### 何时使用 Pinia
- 跨多个组件共享的状态
- 需要持久化的状态
- 全局配置信息

### 何时使用组件状态
- 仅在单个组件使用
- 临时的 UI 状态
- 表单数据

### Store 设计

#### 用户 Store
```javascript
// stores/user.js
import { defineStore } from 'pinia';
import { ref, computed } from 'vue';
import { userApi } from '@/api/user';

export const useUserStore = defineStore('user', () => {
  // 状态
  const userInfo = ref(null);
  const token = ref(localStorage.getItem('token') || '');

  // 计算属性
  const isLoggedIn = computed(() => !!token.value);
  const userName = computed(() => userInfo.value?.name || '未登录');
  const userRole = computed(() => userInfo.value?.role || 'guest');

  // 方法
  const login = async (credentials) => {
    const result = await userApi.login(credentials);
    token.value = result.token;
    userInfo.value = result.user;
    localStorage.setItem('token', result.token);
  };

  const logout = () => {
    token.value = '';
    userInfo.value = null;
    localStorage.removeItem('token');
  };

  const fetchUserInfo = async () => {
    if (!token.value) return;
    userInfo.value = await userApi.getUserInfo();
  };

  return {
    userInfo,
    token,
    isLoggedIn,
    userName,
    userRole,
    login,
    logout,
    fetchUserInfo
  };
});
```

#### 应用 Store
```javascript
// stores/app.js
import { defineStore } from 'pinia';
import { ref } from 'vue';

export const useAppStore = defineStore('app', () => {
  const sidebarCollapsed = ref(false);
  const theme = ref(localStorage.getItem('theme') || 'light');

  const toggleSidebar = () => {
    sidebarCollapsed.value = !sidebarCollapsed.value;
  };

  const setTheme = (newTheme) => {
    theme.value = newTheme;
    localStorage.setItem('theme', newTheme);
  };

  return {
    sidebarCollapsed,
    theme,
    toggleSidebar,
    setTheme
  };
});
```

## 数据流设计

### 单向数据流
```
父组件 (数据源)
  ↓ props
子组件 (展示)
  ↓ emit
父组件 (处理)
```

### Props Down, Events Up
```vue
<!-- 父组件 -->
<script setup>
const taskList = ref([]);

const handleTaskUpdate = (updatedTask) => {
  const index = taskList.value.findIndex(t => t.id === updatedTask.id);
  if (index !== -1) {
    taskList.value[index] = updatedTask;
  }
};
</script>

<template>
  <TaskItem
    v-for="task in taskList"
    :key="task.id"
    :task="task"
    @update="handleTaskUpdate"
  />
</template>

<!-- 子组件 -->
<script setup>
const props = defineProps({
  task: {
    type: Object,
    required: true
  }
});

const emit = defineEmits(['update']);

const handleEdit = async () => {
  const updated = await taskApi.update(props.task.id, formData);
  emit('update', updated);
};
</script>
```

## 错误边界

### 全局错误处理
```javascript
// main.js
import { createApp } from 'vue';

const app = createApp(App);

app.config.errorHandler = (err, instance, info) => {
  console.error('全局错误:', err);
  console.error('错误信息:', info);
  
  // 上报错误到监控系统
  // reportError(err, info);
};
```

### 组件错误处理
```vue
<script setup>
import { onErrorCaptured } from 'vue';

onErrorCaptured((err, instance, info) => {
  console.error('组件错误:', err);
  return false; // 阻止错误继续传播
});
</script>
```

## 性能优化策略

### 1. 虚拟滚动
- 大列表使用虚拟滚动
- 推荐使用 `vue-virtual-scroller`

### 2. 防抖和节流
```javascript
// composables/useDebounce.js
import { ref } from 'vue';

export const useDebounce = (fn, delay = 300) => {
  let timerId = null;
  
  return (...args) => {
    if (timerId) clearTimeout(timerId);
    timerId = setTimeout(() => {
      fn(...args);
    }, delay);
  };
};
```

### 3. 组件缓存
```vue
<template>
  <router-view v-slot="{ Component }">
    <keep-alive :include="['TaskList', 'TaskDetail']">
      <component :is="Component" />
    </keep-alive>
  </router-view>
</template>
```

### 4. 图片懒加载
```vue
<img 
  v-for="item in list" 
  :key="item.id"
  :src="item.image" 
  loading="lazy"
  alt=""
/>
```

## 测试策略

### 单元测试
- 使用 Vitest
- 测试工具函数
- 测试 composables

```javascript
// utils/format.test.js
import { describe, it, expect } from 'vitest';
import { formatFileSize } from './format';

describe('formatFileSize', () => {
  it('应该正确格式化字节数', () => {
    expect(formatFileSize(0)).toBe('0 B');
    expect(formatFileSize(1024)).toBe('1.00 KB');
    expect(formatFileSize(1048576)).toBe('1.00 MB');
  });
});
```

### 组件测试
- 使用 @vue/test-utils
- 测试组件行为
- 测试用户交互

```javascript
// components/TaskItem.test.js
import { mount } from '@vue/test-utils';
import { describe, it, expect } from 'vitest';
import TaskItem from './TaskItem.vue';

describe('TaskItem', () => {
  it('应该渲染任务名称', () => {
    const wrapper = mount(TaskItem, {
      props: {
        task: { id: '1', name: '测试任务' }
      }
    });
    expect(wrapper.text()).toContain('测试任务');
  });
});
```

## 代码审查清单

- [ ] 组件职责单一，不超过 300 行
- [ ] 可复用逻辑提取为 composables
- [ ] API 调用封装在 api 目录
- [ ] 全局状态使用 Pinia 管理
- [ ] 组件间通信遵循单向数据流
- [ ] 错误处理完善
- [ ] 性能优化到位（懒加载、虚拟滚动等）
- [ ] 代码有适当的注释
- [ ] 遵循命名规范
- [ ] 通过 ESLint 检查
