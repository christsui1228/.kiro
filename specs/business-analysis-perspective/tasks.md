# 业务分析功能升级任务清单（Perspective 4.0 架构）

## 任务概述

将现有的业务分析功能升级为基于 Perspective 4.0 + ECharts 6.0 的现代化数据分析平台。

## 后端任务

### 阶段 1：Arrow IPC 支持

- [ ] 1. 修改 Service 层返回原始数据
  - **文件**: `features/business_analysis/services/customer_order_service.py`
  - **需求**: REQ-001
  - **描述**: 修改 Service 层，返回明细数据而非聚合数据
  - **实施步骤**：
    1. 创建 get_raw_data() 方法
    2. 修改 SQL 查询，返回所有订单明细
    3. 添加更多维度列（year, month, quarter, day_of_week）
    4. 移除聚合逻辑（由前端 Perspective 处理）
    5. 直接查询原始表（customers + orders）
  - **验收标准**：
    - 返回明细数据，不做聚合
    - 包含所有必要的维度列
    - 查询性能满足要求（< 2秒）

- [x] 2. 实现 Arrow IPC 导出和 HTTP 缓存
  - **文件**: `features/business_analysis/routes.py`
  - **需求**: REQ-001, REQ-011
  - **描述**: 添加 Arrow IPC 格式的 API 端点，使用 HTTP 缓存
  - **实施步骤**：
    1. 创建 /customer-orders/arrow 端点
    2. 调用 Service 层获取数据
    3. 使用 PyArrow 转换为 Arrow IPC 格式
    4. 生成 ETag（基于数据内容的 MD5）
    5. 检查 If-None-Match 头，支持 304 响应
    6. 设置 Cache-Control 头（max-age=3600）
    7. 设置正确的 Content-Type 头
  - **验收标准**：
    - 返回 Arrow IPC 二进制数据
    - Content-Type 为 application/vnd.apache.arrow.stream
    - ETag 机制正常工作
    - 304 响应正常返回
    - Cache-Control 头设置正确

## 前端任务

### 阶段 2：Perspective 4.0 集成

- [x] 3. 安装 Perspective 4.0 依赖
  - **文件**: `OneManage_web/package.json`
  - **需求**: REQ-002
  - **描述**: 安装 Perspective 4.0 相关包
  - **实施步骤**：
    1. 安装 @finos/perspective@^4.0.0
    2. 安装 @finos/perspective-viewer@^4.0.0
    3. 安装 @finos/perspective-viewer-datagrid@^4.0.0
    4. 安装 @finos/perspective-viewer-d3fc@^4.0.0
    5. 更新 echarts 到 6.0+
  - **验收标准**：
    - 所有依赖安装成功
    - 版本号正确

- [x] 4. 创建 Perspective 主组件
  - **文件**: `OneManage_web/src/views/business-analysis-v2/index.vue`
  - **需求**: REQ-002, REQ-003, REQ-004
  - **描述**: 创建基于 Perspective 的分析页面
  - **实施步骤**：
    1. 创建新目录 business-analysis-v2
    2. 创建 index.vue 主组件
    3. 集成 perspective-viewer 组件
    4. 实现数据加载逻辑（fetch Arrow 数据）
    5. 配置默认视图（Datagrid + 默认分组）
    6. 添加 Loading 状态
    7. 添加错误处理
  - **验收标准**：
    - Perspective Viewer 正常渲染
    - 可以加载 Arrow 数据
    - 默认视图配置正确
    - Loading 和错误状态正常

- [x] 5. 实现数据加载 Composable
  - **文件**: `OneManage_web/src/views/business-analysis-v2/composables/usePerspectiveData.ts`
  - **需求**: REQ-001, REQ-002
  - **描述**: 封装 Arrow 数据加载逻辑
  - **实施步骤**：
    1. 创建 usePerspectiveData composable
    2. 实现 loadArrowData() 方法
    3. 实现错误处理和重试逻辑
    4. 添加加载状态管理
    5. 添加缓存状态显示（从响应头读取）
  - **验收标准**：
    - API 调用正常
    - 错误处理完善
    - 状态管理正确
    - 显示缓存命中状态

- [x] 6. 配置 Perspective 视图选项
  - **文件**: `OneManage_web/src/views/business-analysis-v2/components/PerspectiveConfig.vue`
  - **需求**: REQ-003, REQ-004, REQ-005
  - **描述**: 提供 Perspective 视图配置界面
  - **实施步骤**：
    1. 创建 PerspectiveConfig 组件
    2. 添加视图类型选择（Datagrid, Pivot, Chart）
    3. 添加预设配置（常用分析场景）
    4. 实现配置保存和加载
    5. 添加重置按钮
  - **验收标准**：
    - 视图类型切换正常
    - 预设配置可用
    - 配置可以保存和恢复

### 阶段 3：ECharts 6.0 集成

- [x] 7. 实现 ECharts 集成组件
  - **文件**: `OneManage_web/src/views/business-analysis-v2/components/EChartsPanel.vue`
  - **需求**: REQ-008
  - **描述**: 创建 ECharts 高级图表面板
  - **实施步骤**：
    1. 创建 EChartsPanel 组件
    2. 实现从 Perspective 获取数据的方法
    3. 实现数据格式转换（Perspective → ECharts）
    4. 配置 ECharts 图表选项
    5. 实现手动同步按钮
    6. 实现自动同步开关
    7. 监听 Perspective 视图变化事件
  - **验收标准**：
    - ECharts 正常渲染
    - 可以从 Perspective 获取数据
    - 手动同步和自动同步都正常工作
    - 图表配置合理

- [x] 8. 实现图表模板
  - **文件**: `OneManage_web/src/views/business-analysis-v2/composables/useEChartsTemplates.ts`
  - **需求**: REQ-008
  - **描述**: 提供常用的 ECharts 图表模板
  - **实施步骤**：
    1. 创建 useEChartsTemplates composable
    2. 实现柱状图模板
    3. 实现折线图模板
    4. 实现饼图模板
    5. 实现散点图模板
    6. 实现漏斗图模板
    7. 添加模板选择器
  - **验收标准**：
    - 所有模板可用
    - 模板切换流畅
    - 图表配置合理

### 阶段 4：高级功能

- [ ] 9. 实现数据导出功能
  - **文件**: `OneManage_web/src/views/business-analysis-v2/components/ExportPanel.vue`
  - **需求**: REQ-009
  - **描述**: 实现数据导出功能
  - **实施步骤**：
    1. 创建 ExportPanel 组件
    2. 实现导出为 CSV
    3. 实现导出为 JSON
    4. 实现导出为 Arrow
    5. 实现导出为 Excel（使用 xlsx 库）
    6. 添加导出进度提示
  - **验收标准**：
    - 所有格式导出正常
    - 导出数据包含当前筛选和聚合结果
    - 大数据量导出不卡顿

- [ ] 10. 实现视图保存和恢复
  - **文件**: `OneManage_web/src/views/business-analysis-v2/composables/useViewState.ts`
  - **需求**: 扩展功能
  - **描述**: 实现分析视图的保存和恢复
  - **实施步骤**：
    1. 创建 useViewState composable
    2. 实现 saveView() 方法（保存到 localStorage）
    3. 实现 loadView() 方法
    4. 实现 listViews() 方法
    5. 实现 deleteView() 方法
    6. 添加视图管理界面
  - **验收标准**：
    - 视图可以保存和恢复
    - 视图列表正常显示
    - 可以删除视图

- [x] 11. 添加路由配置
  - **文件**: `OneManage_web/src/router/index.ts`
  - **需求**: REQ-002
  - **描述**: 添加新分析页面的路由
  - **实施步骤**：
    1. 添加 /business-analysis-v2 路由
    2. 配置路由元信息
    3. 添加导航菜单项
    4. 保留旧版本路由（/business-analysis）
  - **验收标准**：
    - 新路由可访问
    - 导航菜单显示正常
    - 旧版本仍然可用

### 阶段 5：优化和测试

- [x] 12. 性能优化
  - **文件**: 多个文件
  - **需求**: REQ-010
  - **描述**: 优化前端性能
  - **实施步骤**：
    1. 实现 Perspective Worker 的懒加载
    2. 优化 WebAssembly 加载
    3. 实现组件懒加载
    4. 添加性能监控（加载时间、操作响应时间）
    5. 优化大数据集的渲染
  - **验收标准**：
    - 首次加载时间 < 3 秒
    - 数据加载时间 < 2 秒（50W 行）
    - 操作响应时间 < 100ms

- [x] 13. 错误处理和用户引导
  - **文件**: `OneManage_web/src/views/business-analysis-v2/components/UserGuide.vue`
  - **需求**: REQ-012
  - **描述**: 完善错误处理和用户引导
  - **实施步骤**：
    1. 创建 UserGuide 组件
    2. 添加首次使用引导
    3. 添加操作提示（Tooltip）
    4. 完善错误提示信息
    5. 添加帮助文档链接
  - **验收标准**：
    - 首次使用有引导
    - 错误提示友好
    - 帮助文档完整

## 测试任务（可选）

- [x] 14. 后端集成测试
  - **文件**: `tests/test_business_analysis_arrow.py`
  - **需求**: REQ-001, REQ-010
  - **描述**: 测试 Arrow IPC 端点
  - **实施步骤**：
    1. 测试 Arrow 数据格式正确性
    2. 测试缓存机制（ETag 和 304 响应）
    3. 测试性能（30W 行 < 2秒）
  - **验收标准**：
    - 所有测试通过
    - 性能测试达标

- [ ] 15. 前端组件测试
  - **文件**: `OneManage_web/src/views/business-analysis-v2/__tests__/`
  - **需求**: REQ-002, REQ-008
  - **描述**: 测试前端组件
  - **实施步骤**：
    1. 测试 Perspective 数据加载
    2. 测试 ECharts 集成
    3. 测试数据导出
    4. 测试视图保存和恢复
  - **验收标准**：
    - 所有测试通过
    - 覆盖率 > 70%

---

## 任务依赖关系

```
后端任务流程：
任务1 → 任务2

前端任务流程：
任务3 → 任务4 → 任务5 → 任务6
任务4 → 任务7 → 任务8
任务4 → 任务9 → 任务10 → 任务11

优化任务：
任务11 → 任务12 → 任务13

测试任务（可选）：
任务2 → 任务14
任务11 → 任务15
```

## 任务优先级

**P0（核心功能）**：
- 任务 1-2（后端 Arrow 支持）
- 任务 3-6（Perspective 集成）
- 任务 11（路由配置）

**P1（增强功能）**：
- 任务 7-8（ECharts 集成）
- 任务 9-10（高级功能）
- 任务 12-13（优化）

**P2（可选功能）**：
- 任务 14-15（测试）

## 预估工作量

- 后端任务（任务 1-2）：1-2 天
- 前端核心（任务 3-6）：3-4 天
- 前端增强（任务 7-11）：2-3 天
- 优化和测试（任务 12-15）：2 天（可选）

**总计**：6-9 天（不含测试）

## 验收标准总览

### 功能验收
- ✅ 用户可以加载 Arrow 格式数据
- ✅ 用户可以自由拖拽列进行分组
- ✅ 用户可以自由选择聚合方式
- ✅ 用户可以创建 Pivot 透视表
- ✅ 用户可以动态筛选和排序
- ✅ 用户可以切换多种视图类型
- ✅ 用户可以使用 ECharts 创建高级图表
- ✅ 用户可以导出分析结果

### 性能验收
- ✅ 数据加载时间 < 2 秒（50W 行）
- ✅ 前端操作响应 < 100ms
- ✅ 缓存命中响应 < 50ms
- ✅ 虚拟滚动支持 1000W 行

### 代码质量验收
- ✅ 代码结构清晰，符合项目规范
- ✅ 关键逻辑有中文注释
- ✅ 错误处理完善
- ✅ 性能监控完整

## 注意事项

1. **版本要求**：
   - Perspective 必须使用 4.0+ 版本
   - ECharts 必须使用 6.0+ 版本
   - PyArrow 必须使用 20.0+ 版本

2. **浏览器兼容性**：
   - 必须支持 WebAssembly
   - 建议使用 Chrome 90+ 或 Firefox 88+

3. **性能优化**：
   - 使用 HTTP 缓存（ETag + Cache-Control）
   - 前端使用 WebAssembly 加速
   - 数据库查询优化（使用已有索引）

4. **渐进式迁移**：
   - 保留旧版本功能
   - 新旧版本并存
   - 逐步引导用户迁移

5. **未来扩展**：
   - 当数据量增长到 100W+ 时，考虑引入物化视图
   - 可以添加 Redis 缓存层进一步优化性能
