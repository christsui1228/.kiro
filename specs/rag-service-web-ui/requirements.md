# RAG 服务 Web 管理界面需求文档

## 简介

为 RAG 智能解析服务开发一个基于 Vue3 + Naive UI 的 Web 管理界面，提供索引管理、智能搜索和知识库数据编辑功能。

## 术语表

- **RAG Service**: RAG 智能解析服务后端，提供自然语言查询解析和商品搜索功能
- **Web UI**: 基于 Vue3 和 Naive UI 的前端管理界面
- **索引重建**: 从 PostgreSQL 重新加载知识库数据并在 Qdrant 中创建向量索引的操作
- **智能搜索**: 用户输入自然语言查询，系统解析后返回匹配商品的功能
- **知识库表**: 包括 slang_to_term_map（俗语映射）、attribute_definitions（属性定义）、relation_rules（关系规则）等数据库表

## 需求

### 需求 1: 索引管理功能

**用户故事**: 作为系统管理员，我希望能够通过 Web 界面重建 RAG 索引，以便在知识库数据更新后刷新向量索引。

#### 验收标准

1. WHEN 管理员点击"重建索引"按钮，THE Web UI SHALL 调用后端 `/api/v1/rebuild-index` 接口
2. WHILE 索引重建进行中，THE Web UI SHALL 显示加载状态指示器
3. WHEN 索引重建成功完成，THE Web UI SHALL 显示成功提示消息
4. IF 索引重建失败，THEN THE Web UI SHALL 显示错误信息和失败原因
5. THE Web UI SHALL 在索引重建完成后自动隐藏加载状态

### 需求 2: 智能搜索功能

**用户故事**: 作为用户，我希望能够在搜索框中输入自然语言查询，以便快速找到符合条件的商品。

#### 验收标准

1. THE Web UI SHALL 提供一个搜索输入框供用户输入查询文本
2. WHEN 用户提交搜索查询，THE Web UI SHALL 调用后端 `/api/v1/search` 接口
3. WHILE 搜索进行中，THE Web UI SHALL 显示加载状态
4. WHEN 搜索成功返回结果，THE Web UI SHALL 以表格或卡片形式展示商品列表
5. THE Web UI SHALL 显示每个商品的关键信息（产品名称、风格、颜色、尺寸、价格等）
6. THE Web UI SHALL 显示搜索结果的总数量
7. THE Web UI SHALL 显示 RAG 解析的查询条件和解释说明
8. IF 搜索失败，THEN THE Web UI SHALL 显示错误提示信息

### 需求 3: 知识库数据编辑功能（后续实现）

**用户故事**: 作为数据管理员，我希望能够通过 Web 界面编辑知识库表数据，以便维护和更新系统的知识库内容。

#### 验收标准

1. THE Web UI SHALL 提供导航菜单访问不同的知识库表管理页面
2. THE Web UI SHALL 支持查看 slang_to_term_map 表的数据列表
3. THE Web UI SHALL 支持查看 attribute_definitions 表的数据列表
4. THE Web UI SHALL 支持查看 relation_rules 表的数据列表
5. WHEN 用户点击"新增"按钮，THE Web UI SHALL 显示数据录入表单
6. WHEN 用户点击"编辑"按钮，THE Web UI SHALL 显示预填充的编辑表单
7. WHEN 用户点击"删除"按钮，THE Web UI SHALL 显示确认对话框
8. WHEN 用户提交表单，THE Web UI SHALL 调用对应的后端 CRUD API
9. WHEN 数据操作成功，THE Web UI SHALL 刷新数据列表并显示成功提示
10. IF 数据操作失败，THEN THE Web UI SHALL 显示错误信息

### 需求 4: 用户界面响应式设计

**用户故事**: 作为用户，我希望界面能够适配不同屏幕尺寸，以便在桌面和移动设备上都能正常使用。

#### 验收标准

1. THE Web UI SHALL 在桌面浏览器（>1024px）上正常显示所有功能
2. THE Web UI SHALL 在平板设备（768px-1024px）上自适应布局
3. THE Web UI SHALL 在移动设备（<768px）上提供简化的操作界面
4. THE Web UI SHALL 使用 Naive UI 的响应式组件确保跨设备兼容性

### 需求 5: API 通信和错误处理

**用户故事**: 作为开发者，我希望前端能够正确处理 API 通信和各种错误情况，以便提供稳定可靠的用户体验。

#### 验收标准

1. THE Web UI SHALL 使用 Axios 或 Fetch API 与后端通信
2. THE Web UI SHALL 在请求头中设置正确的 Content-Type
3. WHEN API 返回 4xx 错误，THE Web UI SHALL 显示用户友好的错误提示
4. WHEN API 返回 5xx 错误，THE Web UI SHALL 显示服务器错误提示
5. IF 网络连接失败，THEN THE Web UI SHALL 显示网络错误提示
6. THE Web UI SHALL 为所有 API 请求设置合理的超时时间（30秒）
