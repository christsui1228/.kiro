# RAG 服务 Web 管理界面实施计划

## 任务列表

- [ ] 1. 初始化 Vue3 项目和配置开发环境
  - 使用 Vite 创建 Vue3 + TypeScript 项目
  - 安装 Naive UI、Axios、VueUse 等依赖
  - 配置 TypeScript、Vite、路径别名
  - 创建项目目录结构（api、components、types、composables、views）
  - 配置环境变量文件（.env.development）
  - _需求: 1.1, 2.1, 5.1_

- [ ] 2. 实现 API 层和类型定义
  - [ ] 2.1 创建 TypeScript 类型定义
    - 定义 ProductResponse、SearchResponse、QueryCondition 等接口
    - 定义 RebuildIndexResponse、ApiError 接口
    - _需求: 2.4, 2.5, 5.2_
  
  - [ ] 2.2 配置 Axios 实例和拦截器
    - 创建 Axios 实例，配置 baseURL 和 timeout
    - 实现请求拦截器（添加通用请求头）
    - 实现响应拦截器（统一错误处理）
    - 集成 Naive UI 的 message 组件显示错误提示
    - _需求: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6_
  
  - [ ] 2.3 实现搜索相关 API 接口
    - 实现 searchProducts 函数（POST /api/v1/search）
    - 实现 rebuildIndex 函数（POST /api/v1/rebuild-index）
    - 实现 debugKnowledge 函数（GET /api/v1/debug/knowledge）
    - _需求: 2.2, 1.1_

- [ ] 3. 实现业务逻辑 Composables
  - [ ] 3.1 实现 useSearch composable
    - 创建响应式状态（loading、searchResult）
    - 实现 search 函数，调用 searchProducts API
    - 实现输入验证和错误处理
    - 集成 Naive UI message 显示操作反馈
    - _需求: 2.1, 2.2, 2.3, 2.4, 2.8_
  
  - [ ] 3.2 实现 useIndex composable
    - 创建响应式状态（rebuilding）
    - 实现 rebuild 函数，调用 rebuildIndex API
    - 集成 Naive UI dialog 显示确认对话框
    - 实现成功/失败消息提示
    - _需求: 1.1, 1.2, 1.3, 1.4, 1.5_

- [ ] 4. 实现核心 UI 组件
  - [ ] 4.1 实现 SearchBox 组件
    - 使用 n-input 创建搜索输入框
    - 使用 n-button 创建搜索按钮
    - 实现 loading 状态显示
    - 定义 props（placeholder、loading）和 emits（search）
    - 添加输入防抖处理（可选优化）
    - _需求: 2.1, 2.3_
  
  - [ ] 4.2 实现 ProductTable 组件
    - 使用 n-data-table 创建商品表格
    - 定义表格列（产品编码、风格、价格、重量、材质、等级、特性）
    - 使用 n-tag 展示材质列表
    - 使用 n-card 展示查询解释
    - 使用 n-collapse 展示查询条件详情
    - 实现 loading 状态和空状态显示
    - _需求: 2.4, 2.5, 2.6, 2.7_
  
  - [ ] 4.3 实现 IndexManager 组件
    - 使用 n-card 创建索引管理卡片
    - 使用 n-button 创建重建索引按钮
    - 使用 n-alert 显示操作提示信息
    - 集成 useIndex composable
    - 实现 loading 状态显示
    - _需求: 1.1, 1.2, 1.3, 1.4, 1.5_

- [ ] 5. 实现主页面视图
  - [ ] 5.1 实现 SearchView 页面
    - 创建页面布局（标题、描述、功能区）
    - 集成 IndexManager 组件
    - 集成 SearchBox 组件
    - 集成 ProductTable 组件
    - 使用 n-space 和 n-layout 组织页面结构
    - 实现响应式布局（使用 n-grid）
    - _需求: 1.1, 2.1, 2.2, 2.4, 4.1, 4.2, 4.3_
  
  - [ ] 5.2 配置 App.vue 根组件
    - 配置 Naive UI 全局组件（n-message-provider、n-dialog-provider）
    - 设置全局主题配置（可选）
    - 集成 SearchView 页面
    - _需求: 1.3, 1.4, 2.8_

- [ ] 6. 完善开发体验和部署配置
  - [ ] 6.1 配置开发服务器
    - 配置 Vite 开发服务器端口
    - 配置 API 代理（proxy）到后端服务
    - 配置热更新和自动刷新
    - _需求: 5.1_
  
  - [ ] 6.2 创建项目文档
    - 编写 README.md（项目介绍、安装、运行、构建）
    - 说明环境变量配置
    - 说明 API 接口对接方式
    - _需求: 所有需求_
  
  - [ ]* 6.3 配置生产构建
    - 配置 Vite 生产构建选项
    - 创建 .env.production 环境变量文件
    - 编写构建脚本
    - _需求: 5.1_

- [ ] 7. 测试和优化
  - [ ]* 7.1 手动功能测试
    - 测试搜索功能的各种输入场景
    - 测试索引重建功能
    - 测试错误处理和边界情况
    - 测试响应式布局在不同设备上的表现
    - _需求: 2.1-2.8, 1.1-1.5, 4.1-4.4, 5.1-5.6_
  
  - [ ]* 7.2 性能优化
    - 检查组件渲染性能
    - 优化大数据量表格显示（虚拟滚动）
    - 添加搜索防抖
    - 检查打包体积和加载速度
    - _需求: 2.3, 2.4_

## 实施说明

### 开发顺序
1. 先搭建项目基础架构（任务 1-2）
2. 实现核心业务逻辑（任务 3）
3. 开发 UI 组件（任务 4）
4. 组装页面（任务 5）
5. 完善配置和文档（任务 6）
6. 测试和优化（任务 7）

### 技术要点
- 使用 Vue 3 Composition API 和 `<script setup>` 语法
- 使用 TypeScript 确保类型安全
- 使用 Naive UI 组件库构建界面
- 使用 Axios 进行 HTTP 通信
- 遵循响应式设计原则

### 测试方式
- 启动后端服务：`cd rag_service && pdm run uvicorn rag_service.main:app --reload`
- 启动前端服务：`cd rag-service-web && npm run dev`
- 在浏览器中访问 `http://localhost:3000` 进行测试

### 注意事项
- 确保后端服务已启动并可访问
- 注意 CORS 配置（后端需要允许前端域名）
- API 接口路径要与后端保持一致
- 错误处理要覆盖网络错误、4xx、5xx 等各种情况
