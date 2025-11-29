---
inclusion: fileMatch
fileMatchPattern: 'OneManage_web/**/*'
---

# 前端技术栈规范

## 语言和交流规范
- **所有对话**: 必须使用简体中文
- **代码注释**: 必须使用简体中文
- **文档编写**: 必须使用简体中文
- **变量函数名**: 使用英文，但注释用简体中文

## 核心技术栈

### 前端框架
- **框架**: Vue 3 (Composition API)
- **UI 库**: Naive UI
- **构建工具**: Vite
- **包管理器**: pnpm

### 状态管理
- **状态管理**: Pinia
- **本地存储**: localStorage / sessionStorage

### HTTP 客户端
- **HTTP 库**: Axios
- **API 封装**: 统一的 `src/api/` 模块化封装

### 路由
- **路由**: Vue Router 4

### 代码风格
- **组合式 API**: 优先使用 `<script setup>`
- **响应式**: 使用 `ref`、`reactive`、`computed`
- **生命周期**: 使用组合式 API 的生命周期钩子

## 项目结构

```
OneManage_web/
├── src/
│   ├── api/              # API 接口封装
│   ├── components/       # Vue 组件
│   ├── stores/           # Pinia 状态管理
│   ├── router/           # 路由配置
│   ├── utils/            # 工具函数
│   └── main.js           # 入口文件
├── public/               # 静态资源
└── package.json
```

## API 调用规范

### API 模块化
```javascript
// src/api/imageGenerator.js
import http from '@/utils/http';

export const imageGeneratorApi = {
  async getLayersPreview(taskId) {
    return await http.get(`/api/image-generator/psd-upload-oss/tasks/${taskId}/layers-preview`);
  }
};
```

### 组件中使用
```vue
<script setup>
import { ref } from 'vue';
import { imageGeneratorApi } from '@/api/imageGenerator';

const layers = ref([]);

const fetchPreview = async (taskId) => {
  try {
    const result = await imageGeneratorApi.getLayersPreview(taskId);
    layers.value = result.layers_preview || [];
  } catch (error) {
    console.error('获取预览失败:', error);
  }
};
</script>
```

## 命名规范

### 文件命名
- **组件**: PascalCase (如 `PsdCanvasPreview.vue`)
- **API 模块**: camelCase (如 `imageGenerator.js`)
- **工具函数**: camelCase (如 `http.js`)

### 变量命名
- **响应式变量**: camelCase (如 `const layers = ref([])`)
- **常量**: UPPER_SNAKE_CASE (如 `const API_BASE_URL = '...'`)
- **组件 props**: camelCase (如 `selectedLayerIndex`)

## 类型安全

虽然项目使用 JavaScript，但应该：
- 在注释中说明参数类型
- 使用 JSDoc 提供类型提示
- 验证 API 返回的数据格式

```javascript
/**
 * 获取图层预览数据
 * @param {string} taskId - 任务 ID
 * @returns {Promise<Object>} 预览数据
 */
async getLayersPreview(taskId) {
  // ...
}
```

## 环境变量配置

### 环境文件
```
.env                # 所有环境通用
.env.development    # 开发环境
.env.production     # 生产环境
```

### 变量命名
- 必须以 `VITE_` 开头
- 使用 UPPER_SNAKE_CASE

```bash
# .env.development
VITE_API_BASE_URL=http://localhost:8000
VITE_APP_TITLE=OneManage开发环境
```

### 使用环境变量
```javascript
const apiBaseUrl = import.meta.env.VITE_API_BASE_URL;
const appTitle = import.meta.env.VITE_APP_TITLE;
```

## HTTP 客户端配置

### Axios 实例配置
```javascript
// src/utils/http.js
import axios from 'axios';
import { useUserStore } from '@/stores/user';

const http = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json'
  }
});

// 请求拦截器
http.interceptors.request.use(
  (config) => {
    const userStore = useUserStore();
    if (userStore.token) {
      config.headers.Authorization = `Bearer ${userStore.token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// 响应拦截器
http.interceptors.response.use(
  (response) => {
    return response.data;
  },
  (error) => {
    if (error.response?.status === 401) {
      const userStore = useUserStore();
      userStore.clearAuth();
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default http;
```

## 路由配置

### 路由定义
```javascript
// src/router/index.js
import { createRouter, createWebHistory } from 'vue-router';

const routes = [
  {
    path: '/',
    name: 'Home',
    component: () => import('@/views/Home.vue')
  },
  {
    path: '/tasks',
    name: 'TaskList',
    component: () => import('@/views/TaskList.vue')
  },
  {
    path: '/tasks/:id',
    name: 'TaskDetail',
    component: () => import('@/views/TaskDetail.vue')
  }
];

const router = createRouter({
  history: createWebHistory(),
  routes
});

export default router;
```

### 路由守卫
```javascript
// 全局前置守卫
router.beforeEach((to, from, next) => {
  const userStore = useUserStore();
  
  // 需要登录的页面
  if (to.meta.requiresAuth && !userStore.isLoggedIn) {
    next('/login');
    return;
  }
  
  next();
});
```

## 构建配置

### Vite 配置
```javascript
// vite.config.js
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import path from 'path';

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src')
    }
  },
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true
      }
    }
  },
  build: {
    outDir: 'dist',
    sourcemap: false,
    rollupOptions: {
      output: {
        manualChunks: {
          'naive-ui': ['naive-ui'],
          'vue-vendor': ['vue', 'vue-router', 'pinia']
        }
      }
    }
  }
});
```

## 开发工具

### VSCode 推荐插件
- Vue Language Features (Volar)
- ESLint
- Prettier
- Auto Rename Tag
- Path Intellisense

### ESLint 配置
```javascript
// .eslintrc.js
module.exports = {
  extends: [
    'plugin:vue/vue3-recommended',
    'eslint:recommended'
  ],
  rules: {
    'vue/multi-word-component-names': 'off',
    'vue/no-v-html': 'off',
    'no-console': process.env.NODE_ENV === 'production' ? 'warn' : 'off',
    'no-debugger': process.env.NODE_ENV === 'production' ? 'error' : 'off'
  }
};
```

### Prettier 配置
```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "none",
  "printWidth": 100
}
```

## 性能优化

### 路由懒加载
```javascript
// ✅ 推荐
const TaskList = () => import('@/views/TaskList.vue');

// ❌ 避免
import TaskList from '@/views/TaskList.vue';
```

### 组件懒加载
```javascript
import { defineAsyncComponent } from 'vue';

const HeavyComponent = defineAsyncComponent(() =>
  import('@/components/HeavyComponent.vue')
);
```

### 图片优化
- 使用 WebP 格式
- 添加 loading="lazy"
- 使用 CDN

```vue
<img 
  src="image.webp" 
  loading="lazy" 
  alt="描述"
/>
```

## 调试技巧

### Vue DevTools
- 安装 Vue DevTools 浏览器插件
- 查看组件树和状态
- 调试 Pinia store

### 日志输出
```javascript
// 开发环境输出日志
if (import.meta.env.DEV) {
  console.log('调试信息:', data);
}
```

## 部署规范

### 构建命令
```bash
# 开发环境
pnpm dev

# 生产构建
pnpm build

# 预览构建结果
pnpm preview
```

### 静态资源
- 放在 `public/` 目录
- 使用绝对路径引用
- 不会被 Vite 处理

```html
<!-- 引用 public/logo.png -->
<img src="/logo.png" alt="Logo" />
```
