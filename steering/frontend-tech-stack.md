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
