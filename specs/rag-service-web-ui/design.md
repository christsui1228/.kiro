# RAG 服务 Web 管理界面设计文档

## 概述

本文档描述 RAG 服务 Web 管理界面的技术设计，包括前端架构、组件设计、数据流和 API 集成方案。

## 技术栈

### 前端框架
- **Vue 3.4+**: 使用 Composition API 和 `<script setup>` 语法
- **Vite 5+**: 构建工具，提供快速的开发体验
- **TypeScript 5+**: 类型安全的开发体验
- **Pinia**: 状态管理（如需要）

### UI 组件库
- **Naive UI**: 主要 UI 组件库
  - 提供完整的组件生态（Table, Form, Button, Message, Dialog 等）
  - 支持主题定制
  - TypeScript 友好

### HTTP 客户端
- **Axios**: HTTP 请求库
  - 支持请求/响应拦截器
  - 自动 JSON 转换
  - 错误处理

### 其他依赖
- **Vue Router 4**: 路由管理（如需要多页面）
- **VueUse**: Vue 组合式工具库

## 项目结构

```
rag-service-web/
├── public/                 # 静态资源
├── src/
│   ├── api/               # API 接口定义
│   │   ├── index.ts       # Axios 实例配置
│   │   ├── search.ts      # 搜索相关 API
│   │   └── knowledge.ts   # 知识库管理 API（后续）
│   ├── components/        # 可复用组件
│   │   ├── SearchBox.vue  # 搜索框组件
│   │   ├── ProductTable.vue  # 商品表格组件
│   │   └── IndexManager.vue  # 索引管理组件
│   ├── types/             # TypeScript 类型定义
│   │   ├── api.ts         # API 响应类型
│   │   └── models.ts      # 数据模型类型
│   ├── views/             # 页面视图
│   │   ├── SearchView.vue # 搜索页面
│   │   └── KnowledgeView.vue # 知识库管理页面（后续）
│   ├── composables/       # 组合式函数
│   │   ├── useSearch.ts   # 搜索逻辑
│   │   └── useIndex.ts    # 索引管理逻辑
│   ├── App.vue            # 根组件
│   └── main.ts            # 应用入口
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
└── README.md
```

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────┐
│           Vue 3 Application             │
│  ┌───────────────────────────────────┐  │
│  │      Views (页面视图)              │  │
│  │  - SearchView                     │  │
│  │  - KnowledgeView (后续)           │  │
│  └───────────────┬───────────────────┘  │
│                  │                       │
│  ┌───────────────▼───────────────────┐  │
│  │    Components (组件层)             │  │
│  │  - SearchBox                      │  │
│  │  - ProductTable                   │  │
│  │  - IndexManager                   │  │
│  └───────────────┬───────────────────┘  │
│                  │                       │
│  ┌───────────────▼───────────────────┐  │
│  │  Composables (业务逻辑层)          │  │
│  │  - useSearch                      │  │
│  │  - useIndex                       │  │
│  └───────────────┬───────────────────┘  │
│                  │                       │
│  ┌───────────────▼───────────────────┐  │
│  │      API Layer (API 层)            │  │
│  │  - Axios Instance                 │  │
│  │  - Request/Response Interceptors  │  │
│  └───────────────┬───────────────────┘  │
└──────────────────┼─────────────────────┘
                   │
                   │ HTTP/JSON
                   ▼
┌─────────────────────────────────────────┐
│      FastAPI Backend (后端服务)         │
│  - POST /api/v1/search                  │
│  - POST /api/v1/rebuild-index           │
│  - GET  /api/v1/debug/knowledge         │
└─────────────────────────────────────────┘
```

### 数据流设计

#### 搜索流程
```
用户输入查询
    ↓
SearchBox 组件触发 search 事件
    ↓
useSearch composable 调用 API
    ↓
显示 loading 状态
    ↓
API 返回结果
    ↓
更新响应式数据
    ↓
ProductTable 组件渲染结果
```

#### 索引重建流程
```
用户点击"重建索引"按钮
    ↓
IndexManager 组件调用 useIndex
    ↓
显示确认对话框
    ↓
用户确认
    ↓
useIndex composable 调用 API
    ↓
显示 loading 状态
    ↓
API 返回结果
    ↓
显示成功/失败消息
    ↓
更新 UI 状态
```

## 组件设计

### 1. SearchBox 组件

**功能**: 提供搜索输入框和搜索按钮

**Props**:
```typescript
interface SearchBoxProps {
  placeholder?: string
  loading?: boolean
}
```

**Emits**:
```typescript
interface SearchBoxEmits {
  (e: 'search', query: string): void
}
```

**UI 元素**:
- `n-input`: 搜索输入框
- `n-button`: 搜索按钮
- `n-space`: 布局容器

### 2. ProductTable 组件

**功能**: 以表格形式展示商品搜索结果

**Props**:
```typescript
interface ProductTableProps {
  products: ProductResponse[]
  loading?: boolean
  explanation?: string
  conditions?: any[]
}
```

**UI 元素**:
- `n-data-table`: 数据表格
- `n-tag`: 材质标签
- `n-card`: 查询解释卡片
- `n-collapse`: 查询条件折叠面板

**表格列定义**:
- 产品编码 (product_code)
- 产品风格 (product_style)
- 价格 (cost)
- 重量 (weight_grams)
- 材质 (materials) - 使用标签展示
- 材质等级 (material_grade)
- 特性 (features)

### 3. IndexManager 组件

**功能**: 提供索引重建功能

**Props**:
```typescript
interface IndexManagerProps {
  // 无需 props
}
```

**UI 元素**:
- `n-card`: 卡片容器
- `n-button`: 重建索引按钮
- `n-modal`: 确认对话框
- `n-alert`: 提示信息

**状态**:
- `rebuilding`: 是否正在重建
- `showConfirm`: 是否显示确认对话框

### 4. SearchView 页面

**功能**: 搜索功能的主页面

**布局**:
```
┌─────────────────────────────────────┐
│         页面标题和描述               │
├─────────────────────────────────────┤
│      IndexManager 组件               │
├─────────────────────────────────────┤
│      SearchBox 组件                  │
├─────────────────────────────────────┤
│      ProductTable 组件               │
│      (显示搜索结果)                  │
└─────────────────────────────────────┘
```

## 数据模型

### TypeScript 类型定义

```typescript
// types/models.ts

/** 商品响应模型 */
export interface ProductResponse {
  id: number
  product_code: string
  product_style: string
  cost: number
  weight_grams: number
  materials: string[]
  material_grade: string | null
  features: string | null
}

/** 搜索响应模型 */
export interface SearchResponse {
  query: string
  conditions: QueryCondition[]
  explanation: string
  products: ProductResponse[]
  total: number
}

/** 查询条件 */
export interface QueryCondition {
  field: string
  operator: string
  value: any
  relation?: string
}

/** 索引重建响应 */
export interface RebuildIndexResponse {
  message: string
  detail: string
}

/** API 错误响应 */
export interface ApiError {
  detail: string
}
```

## API 集成

### Axios 配置

```typescript
// api/index.ts

import axios from 'axios'
import { useMessage } from 'naive-ui'

const message = useMessage()

// 创建 Axios 实例
export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:8000',
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  },
})

// 请求拦截器
apiClient.interceptors.request.use(
  (config) => {
    // 可以在这里添加认证 token
    return config
  },
  (error) => {
    return Promise.reject(error)
  }
)

// 响应拦截器
apiClient.interceptors.response.use(
  (response) => {
    return response
  },
  (error) => {
    // 统一错误处理
    if (error.response) {
      const status = error.response.status
      const detail = error.response.data?.detail || '请求失败'
      
      if (status >= 500) {
        message.error(`服务器错误: ${detail}`)
      } else if (status >= 400) {
        message.error(`请求错误: ${detail}`)
      }
    } else if (error.request) {
      message.error('网络连接失败，请检查网络')
    } else {
      message.error('请求配置错误')
    }
    
    return Promise.reject(error)
  }
)
```

### API 接口定义

```typescript
// api/search.ts

import { apiClient } from './index'
import type { SearchResponse, RebuildIndexResponse } from '@/types/models'

/** 搜索商品 */
export const searchProducts = async (query: string): Promise<SearchResponse> => {
  const response = await apiClient.post<SearchResponse>('/api/v1/search', {
    query,
  })
  return response.data
}

/** 重建索引 */
export const rebuildIndex = async (): Promise<RebuildIndexResponse> => {
  const response = await apiClient.post<RebuildIndexResponse>('/api/v1/rebuild-index')
  return response.data
}

/** 调试知识库 */
export const debugKnowledge = async (slang?: string) => {
  const response = await apiClient.get('/api/v1/debug/knowledge', {
    params: { slang },
  })
  return response.data
}
```

## Composables 设计

### useSearch

```typescript
// composables/useSearch.ts

import { ref } from 'vue'
import { useMessage } from 'naive-ui'
import { searchProducts } from '@/api/search'
import type { SearchResponse } from '@/types/models'

export function useSearch() {
  const message = useMessage()
  
  const loading = ref(false)
  const searchResult = ref<SearchResponse | null>(null)
  
  const search = async (query: string) => {
    if (!query.trim()) {
      message.warning('请输入搜索内容')
      return
    }
    
    loading.value = true
    try {
      const result = await searchProducts(query)
      searchResult.value = result
      message.success(`找到 ${result.total} 个商品`)
    } catch (error) {
      console.error('搜索失败:', error)
      searchResult.value = null
    } finally {
      loading.value = false
    }
  }
  
  return {
    loading,
    searchResult,
    search,
  }
}
```

### useIndex

```typescript
// composables/useIndex.ts

import { ref } from 'vue'
import { useMessage, useDialog } from 'naive-ui'
import { rebuildIndex } from '@/api/search'

export function useIndex() {
  const message = useMessage()
  const dialog = useDialog()
  
  const rebuilding = ref(false)
  
  const rebuild = async () => {
    // 显示确认对话框
    dialog.warning({
      title: '确认重建索引',
      content: '重建索引会删除现有的向量索引并重新创建，此操作可能需要几分钟时间。确定要继续吗？',
      positiveText: '确定',
      negativeText: '取消',
      onPositiveClick: async () => {
        rebuilding.value = true
        try {
          const result = await rebuildIndex()
          message.success(result.message)
        } catch (error) {
          console.error('索引重建失败:', error)
        } finally {
          rebuilding.value = false
        }
      },
    })
  }
  
  return {
    rebuilding,
    rebuild,
  }
}
```

## 样式设计

### 主题配置

使用 Naive UI 的主题定制功能：

```typescript
// main.ts

import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

// Naive UI 主题配置
const themeOverrides = {
  common: {
    primaryColor: '#18a058',
    primaryColorHover: '#36ad6a',
    primaryColorPressed: '#0c7a43',
  },
}

app.mount('#app')
```

### 响应式布局

使用 Naive UI 的 Grid 系统实现响应式布局：

```vue
<n-grid :cols="24" :x-gap="12">
  <n-grid-item :span="24" :md="18" :lg="16">
    <!-- 主内容区 -->
  </n-grid-item>
  <n-grid-item :span="24" :md="6" :lg="8">
    <!-- 侧边栏 -->
  </n-grid-item>
</n-grid>
```

## 错误处理策略

### API 错误处理

1. **网络错误**: 显示"网络连接失败"提示
2. **4xx 错误**: 显示具体的错误信息
3. **5xx 错误**: 显示"服务器错误"提示
4. **超时错误**: 显示"请求超时"提示

### 用户反馈

- 使用 `n-message` 显示操作结果提示
- 使用 `n-dialog` 显示确认对话框
- 使用 `n-spin` 显示加载状态
- 使用 `n-empty` 显示空状态

## 性能优化

### 前端优化

1. **组件懒加载**: 使用 Vue 的异步组件
2. **防抖处理**: 搜索输入使用防抖
3. **虚拟滚动**: 大量数据使用虚拟滚动（Naive UI 的 `n-data-table` 支持）
4. **代码分割**: Vite 自动进行代码分割

### 缓存策略

1. **搜索结果缓存**: 可选，使用 Map 缓存最近的搜索结果
2. **API 响应缓存**: 使用 Axios 的缓存适配器（可选）

## 开发环境配置

### 环境变量

```bash
# .env.development
VITE_API_BASE_URL=http://localhost:8000

# .env.production
VITE_API_BASE_URL=https://api.yourdomain.com
```

### 开发服务器配置

```typescript
// vite.config.ts

import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { resolve } from 'path'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      },
    },
  },
})
```

## 部署方案

### 构建配置

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc && vite build",
    "preview": "vite preview"
  }
}
```

### 静态文件部署

构建后的文件可以部署到：
- Nginx
- Apache
- 阿里云 OSS + CDN
- Vercel / Netlify

### Docker 部署（可选）

```dockerfile
FROM node:20-alpine as builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## 测试策略

### 单元测试（可选）

- 使用 Vitest 进行组件测试
- 测试 composables 的业务逻辑
- 测试 API 调用（使用 Mock）

### E2E 测试（可选）

- 使用 Playwright 或 Cypress
- 测试关键用户流程

## 后续扩展

### 知识库管理功能

当后端实现 CRUD API 后，前端需要添加：

1. **数据表格组件**: 支持增删改查
2. **表单组件**: 数据录入和编辑
3. **分页组件**: 处理大量数据
4. **搜索过滤**: 表格内搜索

### 用户认证（可选）

如果需要用户认证：

1. 添加登录页面
2. 实现 JWT token 管理
3. 在 Axios 拦截器中添加 token
4. 实现路由守卫

## 技术决策记录

### 为什么选择 Naive UI？

1. **TypeScript 友好**: 完整的类型定义
2. **组件丰富**: 提供所有需要的组件
3. **主题定制**: 支持深度定制
4. **文档完善**: 中文文档，易于上手
5. **性能优秀**: 虚拟滚动等性能优化

### 为什么使用 Composition API？

1. **逻辑复用**: Composables 模式更易复用
2. **类型推导**: TypeScript 支持更好
3. **代码组织**: 按功能组织代码更清晰
4. **Vue 3 推荐**: 官方推荐的新写法

### 为什么选择 Vite？

1. **开发速度**: 极快的冷启动和热更新
2. **现代化**: 原生 ESM 支持
3. **插件生态**: 丰富的插件系统
4. **Vue 官方推荐**: Vue 团队维护
