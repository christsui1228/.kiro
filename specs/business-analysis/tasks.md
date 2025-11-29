# 业务分析功能任务清单

## 任务概述

本任务清单用于实现客户订单分析功能（MVP 版本），包括后端 Polars 分析引擎、FastAPI 接口和前端 Vue 3 界面。

## 后端任务

- [x] 1. 创建模块结构
  - **文件**: `features/business_analysis/__init__.py`
  - **需求**: REQ-001 至 REQ-010
  - **描述**: 创建 business_analysis 模块的基础结构
  - **实施步骤**：
    1. 创建 `features/business_analysis/` 目录
    2. 创建 `__init__.py` 文件
    3. 创建 `constants.py` 定义维度和指标常量
    4. 创建 `schemas.py` 定义 Pydantic 模型
    5. 创建 `analysis.py` 定义分析引擎
    6. 创建 `routes.py` 定义 API 路由
  - **验收标准**：
    - 模块结构清晰，文件命名规范
    - 所有文件包含基本的模块文档字符串

- [x] 2. 定义数据模型
  - **文件**: `features/business_analysis/schemas.py`
  - **需求**: REQ-001, REQ-002, REQ-003
  - **描述**: 使用 Pydantic 定义请求和响应模型
  - **实施步骤**：
    1. 定义 `CustomerOrderAnalysisRequest` 请求模型
       - group_by: 分组维度（customer_source | group_type）
       - time_grain: 时间粒度（year | month | null）
       - date_range: 时间范围（start, end）
    2. 定义 `CustomerOrderAnalysisData` 数据模型
       - dimension_value: 维度值
       - time_period: 时间周期（可选）
       - order_count: 订单数量
       - total_amount: 累计金额
       - avg_amount: 平均金额
    3. 定义 `CustomerOrderAnalysisResponse` 响应模型
       - data: 分析数据列表
       - summary: 汇总信息
       - execution_time: 执行时间
  - **验收标准**：
    - 所有字段有类型注解和描述
    - 包含示例数据（json_schema_extra）
    - 字段验证规则正确（如日期格式）

- [x] 3. 实现 Polars 分析引擎
  - **文件**: `features/business_analysis/analysis.py`
  - **需求**: REQ-004, REQ-005, REQ-008, REQ-009
  - **描述**: 实现基于 Polars 的客户订单分析引擎
  - **实施步骤**：
    1. 创建 `CustomerOrderAnalyzer` 类
    2. 实现 `analyze()` 主方法
       - 接收分析参数
       - 调用内部方法完成分析
       - 返回 Polars DataFrame
    3. 实现 `_load_data()` 方法
       - 构建优化的 SQL 查询
       - 使用 `pl.read_database()` 加载数据
       - 只查询需要的字段
    4. 实现 `_add_time_column()` 方法
       - 根据 time_grain 添加时间列
       - 支持 year 和 month 粒度
    5. 实现 `_aggregate()` 方法
       - 按维度分组
       - 计算订单数量（n_unique）
       - 计算累计金额（sum）
       - 计算平均金额（mean）
       - 格式化金额（保留 2 位小数）
  - **验收标准**：
    - 代码使用 Polars 链式 API
    - 查询性能满足要求（50W 订单 < 5 秒）
    - 计算结果准确（与数据库核对）
    - 包含中文注释说明关键逻辑

- [x] 4. 创建数据库索引
  - **文件**: `alembic/versions/xxx_add_business_analysis_indexes.py`
  - **需求**: REQ-008
  - **描述**: 创建数据库索引以优化查询性能
  - **实施步骤**：
    1. 创建 Alembic 迁移脚本
    2. 添加以下索引：
       - `idx_orders_customer_created` on orders(customer_global_id, created_at)
       - `idx_orders_created_at` on orders(created_at DESC)
       - `idx_customers_source` on customers(customer_source)
       - `idx_customers_group` on customers(group_type)
    3. 使用 `CREATE INDEX CONCURRENTLY` 避免锁表
  - **验收标准**：
    - 迁移脚本可以正常执行
    - 索引创建成功
    - 查询性能有明显提升

- [x] 5. 实现 API 路由
  - **文件**: `features/business_analysis/routes.py`
  - **需求**: REQ-001 至 REQ-010
  - **描述**: 实现 FastAPI 路由接口
  - **实施步骤**：
    1. 创建 APIRouter，prefix="/api/analytics"
    2. 实现 `GET /metadata` 接口
       - 返回可用的维度和时间粒度选项
    3. 实现 `POST /customer-orders` 接口
       - 接收 CustomerOrderAnalysisRequest
       - 验证请求参数（日期范围、维度有效性）
       - 调用 CustomerOrderAnalyzer 执行分析
       - 计算汇总信息
       - 返回 CustomerOrderAnalysisResponse
    4. 添加错误处理
       - 参数错误返回 400
       - 数据为空返回 404
       - 服务器错误返回 500
    5. 添加日志记录（使用 loguru）
  - **验收标准**：
    - API 接口可以正常调用
    - 返回数据格式正确
    - 错误处理完善
    - 响应时间满足要求（< 5 秒）

- [x] 6. 注册路由到主应用
  - **文件**: `main.py`
  - **需求**: REQ-001
  - **描述**: 将业务分析路由注册到 FastAPI 主应用
  - **实施步骤**：
    1. 导入 business_analysis 路由
    2. 使用 `app.include_router()` 注册路由
    3. 验证路由可访问
  - **验收标准**：
    - 路由注册成功
    - 可以通过 `/api/analytics/metadata` 访问
    - Swagger 文档中显示新接口

## 前端任务

- [x] 7. 创建前端模块结构
  - **文件**: `OneManage_web/src/views/business-analysis/`
  - **需求**: REQ-001
  - **描述**: 创建前端业务分析模块的基础结构
  - **实施步骤**：
    1. 创建 `views/business-analysis/` 目录
    2. 创建 `index.vue` 主页面
    3. 创建 `components/` 目录
    4. 创建 `composables/` 目录
  - **验收标准**：
    - 目录结构清晰
    - 文件命名规范

- [x] 8. 实现配置面板组件
  - **文件**: `OneManage_web/src/views/business-analysis/components/ConfigPanel.vue`
  - **需求**: REQ-001, REQ-002, REQ-003
  - **描述**: 使用 Naive UI 实现分析配置面板
  - **实施步骤**：
    1. 创建 ConfigPanel.vue 组件
    2. 使用 `n-radio-group` 实现分组维度选择
       - 客户来源
       - 客户分组
    3. 使用 `n-radio-group` 实现时间粒度选择
       - 不分组
       - 按年
       - 按月
    4. 使用 `n-date-picker` 实现时间范围选择
       - type="daterange"
       - 默认最近一年
    5. 添加"开始分析"和"重置"按钮
    6. 实现配置状态管理（使用 ref）
    7. 实现事件发射（emit analyze 事件）
  - **验收标准**：
    - 界面美观，布局合理
    - 所有配置项可正常操作
    - 默认值设置正确
    - 点击"开始分析"触发分析

- [x] 9. 实现数据表格组件
  - **文件**: `OneManage_web/src/views/business-analysis/components/DataGrid.vue`
  - **需求**: REQ-006
  - **描述**: 使用 Revo Grid 实现数据表格展示
  - **实施步骤**：
    1. 安装 Revo Grid 依赖
       ```bash
       pnpm add @revolist/vue3-datagrid
       ```
    2. 创建 DataGrid.vue 组件
    3. 配置 Revo Grid
       - source: 数据源
       - columns: 列定义（动态生成）
       - theme: "compact"
       - resize: true（支持列宽调整）
       - filter: true（支持列筛选）
    4. 实现动态列配置
       - 维度列（根据 groupBy 显示）
       - 时间周期列（根据 timeGrain 显示）
       - 订单数量列（数字类型）
       - 累计金额列（数字类型）
       - 平均金额列（数字类型）
    5. 实现数据格式化
       - 金额显示千分位分隔符
       - 保留 2 位小数
  - **验收标准**：
    - 表格正常渲染
    - 列配置正确
    - 支持排序和筛选
    - 大数据量流畅滚动

- [x] 10. 实现图表组件
  - **文件**: `OneManage_web/src/views/business-analysis/components/Charts.vue`
  - **需求**: REQ-007
  - **描述**: 使用 ECharts 实现数据可视化
  - **实施步骤**：
    1. 安装 ECharts 依赖
       ```bash
       pnpm add echarts
       ```
    2. 创建 Charts.vue 组件
    3. 使用 `n-tabs` 实现图表切换
       - 柱状图 tab
       - 折线图 tab（仅在有时间粒度时显示）
    4. 实现柱状图
       - X 轴：维度值
       - Y 轴：订单数量和累计金额（双 Y 轴）
       - 支持 tooltip 交互
    5. 实现折线图
       - X 轴：时间周期
       - Y 轴：订单数量和累计金额（双 Y 轴）
       - 显示趋势变化
    6. 实现图表响应式
       - 监听数据变化自动更新
       - 监听窗口大小变化自动调整
  - **验收标准**：
    - 图表正常渲染
    - 数据展示正确
    - 交互流畅
    - 响应式布局正常

- [x] 11. 实现分析逻辑 Composable
  - **文件**: `OneManage_web/src/views/business-analysis/composables/useCustomerOrderAnalysis.ts`
  - **需求**: REQ-001 至 REQ-010
  - **描述**: 实现分析逻辑和状态管理
  - **实施步骤**：
    1. 创建 useCustomerOrderAnalysis.ts
    2. 定义状态
       - config: 分析配置
       - data: 分析结果数据
       - loading: 加载状态
       - error: 错误信息
    3. 实现 API 调用函数
       - fetchMetadata(): 获取元数据
       - analyzeCustomerOrders(): 执行分析
    4. 实现错误处理
       - 网络错误提示
       - 数据为空提示
       - 服务器错误提示
    5. 实现防抖处理
       - 使用 useDebounceFn 延迟 500ms
  - **验收标准**：
    - API 调用正常
    - 状态管理正确
    - 错误处理完善
    - 防抖功能正常

- [x] 12. 实现主页面
  - **文件**: `OneManage_web/src/views/business-analysis/index.vue`
  - **需求**: REQ-001 至 REQ-010
  - **描述**: 组装所有组件，实现完整的分析页面
  - **实施步骤**：
    1. 创建 index.vue 主页面
    2. 导入所有子组件
       - ConfigPanel
       - DataGrid
       - Charts
    3. 使用 useCustomerOrderAnalysis composable
    4. 实现页面布局
       - 顶部：配置面板
       - 中部：数据表格
       - 底部：图表展示
    5. 实现组件间数据传递
       - 配置面板 → 分析逻辑
       - 分析结果 → 表格和图表
    6. 实现 Loading 状态展示
    7. 实现错误提示（使用 n-message）
  - **验收标准**：
    - 页面布局合理
    - 组件协作正常
    - 数据流转正确
    - 用户体验良好

- [x] 13. 添加路由配置
  - **文件**: `OneManage_web/src/router/index.ts`
  - **需求**: REQ-001
  - **描述**: 添加业务分析页面的路由配置
  - **实施步骤**：
    1. 在路由配置中添加新路由
       ```typescript
       {
         path: '/business-analysis',
         name: 'BusinessAnalysis',
         component: () => import('@/views/business-analysis/index.vue'),
         meta: {
           title: '业务分析',
           icon: 'chart-line'
         }
       }
       ```
    2. 在导航菜单中添加入口
  - **验收标准**：
    - 路由配置正确
    - 可以通过 URL 访问页面
    - 导航菜单显示正常

## 测试和验证任务（可选）

- [x] 14. 后端接口测试
  - **文件**: `tests/test_business_analysis.py`
  - **需求**: REQ-008, REQ-009
  - **描述**: 编写后端接口的集成测试
  - **实施步骤**：
    1. 创建测试文件
    2. 测试 metadata 接口
    3. 测试 customer-orders 接口
       - 正常情况
       - 参数错误情况
       - 数据为空情况
    4. 测试性能（50W 订单 < 5 秒）
  - **验收标准**：
    - 所有测试用例通过
    - 性能测试达标

- [x] 15. 前端组件测试
  - **文件**: `OneManage_web/src/views/business-analysis/__tests__/`
  - **需求**: REQ-006, REQ-007
  - **描述**: 编写前端组件的单元测试
  - **实施步骤**：
    1. 测试 ConfigPanel 组件
    2. 测试 DataGrid 组件
    3. 测试 Charts 组件
    4. 测试 useCustomerOrderAnalysis composable
  - **验收标准**：
    - 所有测试用例通过
    - 覆盖率 > 80%

---

## 任务依赖关系

```
后端任务流程：
任务1 → 任务2 → 任务3 → 任务4 → 任务5 → 任务6

前端任务流程：
任务7 → 任务8 ↘
任务7 → 任务9  → 任务11 → 任务12 → 任务13
任务7 → 任务10 ↗

测试任务（可选）：
任务6 → 任务14
任务13 → 任务15
```

## 任务优先级

**P0（核心功能）**：
- 任务 1-6（后端核心）
- 任务 7-13（前端核心）

**P1（可选功能）**：
- 任务 14-15（测试）

## 预估工作量

- 后端任务（任务 1-6）：2-3 天
- 前端任务（任务 7-13）：2-3 天
- 测试任务（任务 14-15）：1 天（可选）

**总计**：4-6 天（不含测试）

## 验收标准总览

### 功能验收
- ✅ 用户可以选择分析维度和时间粒度
- ✅ 用户可以选择时间范围
- ✅ 点击"开始分析"后显示结果
- ✅ 表格正确展示分析数据
- ✅ 图表正确展示数据趋势
- ✅ 错误情况有友好提示

### 性能验收
- ✅ 分析响应时间 < 5 秒（50W 订单）
- ✅ 表格渲染流畅（支持大数据量）
- ✅ 图表渲染时间 < 1 秒

### 代码质量验收
- ✅ 代码结构清晰，符合项目规范
- ✅ 关键逻辑有中文注释
- ✅ 错误处理完善
- ✅ 日志记录完整

## 注意事项

1. **遵循项目规范**：
   - 所有 Python 命令使用 `pdm run` 前缀
   - 代码注释使用简体中文
   - 遵循 Spec 任务范围规范（不包含测试、缓存、监控等）

2. **性能优化**：
   - 数据库查询只查询需要的字段
   - 使用 Polars 链式 API
   - 前端使用虚拟滚动和防抖

3. **错误处理**：
   - 后端统一错误响应格式
   - 前端友好的错误提示
   - 日志记录关键操作

4. **扩展性**：
   - 代码设计考虑未来扩展
   - 维度和指标易于添加
   - 组件可复用
