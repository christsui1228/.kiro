# 业务分析功能升级需求文档（Perspective 4.0 架构）

## 功能概述

将现有的业务分析功能升级为基于 **Perspective 4.0 + ECharts 6.0** 的现代化数据分析平台，实现全链路列式存储、零拷贝数据传输和前端动态分析能力。

## 业务目标

- 实现用户自由探索数据的能力（无需预设维度）
- 提升分析响应速度（从 1-2 秒降至毫秒级）
- 支持复杂的多维分析（Pivot、动态聚合、筛选）
- 降低后端负载（前端承担大部分计算）
- 提供 Power BI 级别的用户体验

## 术语表

- **Perspective**: FINOS 开源的高性能数据可视化引擎，基于 WebAssembly
- **Arrow IPC**: Apache Arrow 的进程间通信格式，支持零拷贝数据传输
- **列式存储**: 按列存储数据的格式，适合分析型查询
- **Pivot 透视**: 数据透视表，支持行列转换和多维分析
- **零拷贝**: 数据在传输过程中不需要序列化/反序列化

## 验收标准（EARS 格式）

### REQ-001: Arrow IPC 数据传输

**WHEN** 用户请求分析数据  
**THE system SHALL** 返回 Arrow IPC 格式的二进制数据

**验收标准**：
1. 后端支持导出 Polars DataFrame 为 Arrow IPC 格式
2. HTTP 响应头设置为 `application/vnd.apache.arrow.stream`
3. 数据传输大小比 JSON 格式减少 40-60%
4. 支持流式传输（Streaming IPC）

### REQ-002: Perspective 4.0 集成

**WHEN** 前端接收到 Arrow 数据  
**THE system SHALL** 使用 Perspective 4.0 加载和渲染数据

**验收标准**：
1. 使用 `@finos/perspective` 4.x 版本
2. 使用 `@finos/perspective-viewer` 4.x 版本
3. 支持 WebAssembly 加载
4. 数据加载时间 < 500ms（50W 行数据）

### REQ-003: 动态维度分组

**WHEN** 用户拖拽列到分组区域  
**THE system SHALL** 即时更新分析结果

**验收标准**：
1. 支持任意列作为分组维度
2. 支持多维度分组（最多 5 个维度）
3. 响应时间 < 100ms
4. 无需后端参与

### REQ-004: 动态聚合计算

**WHEN** 用户改变聚合方式  
**THE system SHALL** 即时重新计算结果

**验收标准**：
1. 支持 sum、avg、count、min、max、median 等聚合函数
2. 支持 distinct count（去重计数）
3. 支持 weighted mean（加权平均）
4. 响应时间 < 100ms

### REQ-005: Pivot 透视表

**WHEN** 用户拖拽列到列分组区域  
**THE system SHALL** 生成透视表

**验收标准**：
1. 支持行分组和列分组
2. 支持多级透视（行和列各最多 3 级）
3. 支持展开/折叠功能
4. 响应时间 < 200ms

### REQ-006: 动态筛选

**WHEN** 用户添加筛选条件  
**THE system SHALL** 即时过滤数据

**验收标准**：
1. 支持数值比较（>、<、=、!=、between）
2. 支持字符串匹配（contains、starts with、ends with）
3. 支持日期范围筛选
4. 支持多条件组合（AND/OR）
5. 响应时间 < 50ms

### REQ-007: 多视图支持

**WHEN** 用户切换视图类型  
**THE system SHALL** 使用对应的渲染方式

**验收标准**：
1. 支持 Datagrid（数据表格）
2. 支持 Pivot Table（透视表）
3. 支持 Y Line（折线图）
4. 支持 Y Bar（柱状图）
5. 支持 X/Y Scatter（散点图）
6. 视图切换时间 < 300ms

### REQ-008: ECharts 6.0 集成

**WHEN** 用户需要高级图表  
**THE system SHALL** 提供 ECharts 6.0 渲染选项

**验收标准**：
1. 使用 ECharts 6.x 版本
2. 支持从 Perspective 数据源驱动 ECharts
3. 支持实时数据同步
4. 支持自定义图表配置

### REQ-009: 数据导出

**WHEN** 用户点击导出按钮  
**THE system SHALL** 导出当前视图数据

**验收标准**：
1. 支持导出为 CSV 格式
2. 支持导出为 JSON 格式
3. 支持导出为 Arrow 格式
4. 导出数据包含当前筛选和聚合结果

### REQ-010: 性能要求

**WHEN** 系统处理大数据集  
**THE system SHALL** 保持流畅的用户体验

**验收标准**：
1. 50W 行数据加载时间 < 2 秒
2. 100W 行数据加载时间 < 5 秒
3. 前端操作响应时间 < 100ms
4. 虚拟滚动支持 1000W 行数据
5. 内存占用 < 500MB（100W 行数据）

### REQ-011: 浏览器缓存策略

**WHEN** 用户重复查询相同数据  
**THE system SHALL** 使用 HTTP 缓存加速响应

**验收标准**：
1. 设置合适的 Cache-Control 响应头
2. 使用 ETag 支持条件请求
3. 浏览器缓存命中时无需网络请求
4. 缓存时间：1 小时（可配置）

### REQ-012: 错误处理

**WHEN** 分析过程中发生错误  
**THE system SHALL** 显示友好的错误提示

**验收标准**：
1. WebAssembly 加载失败：提示浏览器不支持
2. 数据加载失败：提示网络错误并支持重试
3. 内存不足：提示数据量过大，建议筛选
4. 所有错误都有明确的解决建议

## 功能范围

### 包含功能
- ✅ Arrow IPC 数据传输
- ✅ Perspective 4.0 数据加载和渲染
- ✅ 动态维度分组（任意列）
- ✅ 动态聚合计算（sum、avg、count 等）
- ✅ Pivot 透视表
- ✅ 动态筛选和排序
- ✅ 多视图支持（表格、透视、图表）
- ✅ ECharts 6.0 高级图表
- ✅ 数据导出（CSV、JSON、Arrow）
- ✅ HTTP 缓存（浏览器缓存）

### 不包含功能（后续版本）
- ❌ 实时数据流（WebSocket）
- ❌ 协作功能（多用户同时分析）
- ❌ 报表保存和分享
- ❌ 权限控制
- ❌ 自定义计算字段（公式编辑器）
- ❌ AI 驱动的洞察

## 数据来源

### 数据表
- `customers` 表：客户基础信息（~10W 行）
- `orders` 表：订单信息（~30W 行）
- `customer_groups` 表：客户分组信息

### 数据关系
- customers 与 orders 通过 customer_nickname 关联
- customers 与 customer_groups 通过 group_id 关联

## 非功能性需求

### 性能要求
- Arrow 数据加载：< 2 秒（50W 行）
- 前端操作响应：< 100ms
- 缓存命中响应：< 50ms
- 虚拟滚动：支持 1000W 行

### 可用性要求
- 界面直观，支持拖拽操作
- 提供操作提示和引导
- 错误提示友好，有解决建议

### 兼容性要求
- 支持现代浏览器（Chrome 90+、Firefox 88+、Safari 14+、Edge 90+）
- 支持 WebAssembly
- 响应式布局，支持 1920x1080 及以上分辨率

### 可维护性要求
- 代码结构清晰，遵循项目规范
- 关键逻辑有注释说明
- 易于扩展新的数据源和视图类型

## 约束条件

### 技术约束
- 后端使用 FastAPI + Polars + PyArrow
- 前端使用 Vue 3 + Perspective 4.0 + ECharts 6.0
- 数据库为 PostgreSQL（不能安装自定义扩展）
- 使用 HTTP 缓存（无需 Redis）

### 业务约束
- 暂不考虑权限控制
- 暂不支持多店铺数据隔离
- 数据实时性要求：实时查询数据库

### 数据约束
- 订单数据量当前：30W 条
- 客户数据量当前：10W 条
- 单次查询数据量限制：最多 50W 行

## 成功指标

- 用户操作响应时间 < 100ms（达成率 > 95%）
- 数据加载时间 < 2 秒（50W 行数据）
- 缓存命中率 > 80%
- 用户满意度 > 4.5/5
- 系统可用性 > 99.5%
