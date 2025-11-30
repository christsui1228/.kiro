# 业务分析混合架构任务清单

## 任务概述

实现基于混合架构的业务分析功能，将数据探索和业务分析模型分离为两个独立的功能模块。

## 后端任务

### 阶段 1：Service 层重构

- [x] 1. 重构 Service 层支持两种数据流
  - **文件**: `features/business_analysis/services/customer_order_service.py`
  - **需求**: REQ-A002, REQ-B003
  - **描述**: 重构 Service 层，支持返回原始数据和聚合数据
  - **实施步骤**：
    1. 创建 `get_raw_data()` 方法（用于 Perspective）
       - 查询原始数据（JOIN customers + orders）
       - 添加时间维度列（year, month, quarter, day_of_week）
       - 返回 Polars DataFrame
    2. 创建 `analyze()` 方法（用于 ECharts）
       - 调用 `get_raw_data()` 获取原始数据
       - 根据 `group_by` 和 `time_grain` 进行聚合
       - 计算订单数量、累计金额、平均金额
       - 格式化金额（保留 2 位小数）
       - 返回 Polars DataFrame
    3. 优化数据库查询
       - 只查询需要的列
       - 使用 INNER JOIN
       - 利用已有索引
  - **验收标准**：
    - `get_raw_data()` 返回明细数据，包含所有维度列
    - `analyze()` 返回聚合数据，计算正确
    - 查询性能满足要求（< 2 秒，50W 行）

- [ ]* 1.1 编写 Service 层属性测试
  - **属性 3: 聚合结果正确性**
  - **验证需求: REQ-B003**


### 阶段 2：API 接口实现

- [x] 2. 实现 Arrow 数据接口
  - **文件**: `features/business_analysis/routes.py`
  - **需求**: REQ-A002
  - **描述**: 创建 Arrow IPC 格式的数据接口
  - **实施步骤**：
    1. 创建 `GET /customer-orders/arrow` 端点
    2. 接收日期范围参数
    3. 调用 Service 层 `get_raw_data()` 方法
    4. 使用 PyArrow 转换为 Arrow IPC 格式
    5. 生成 ETag（基于数据内容的 MD5）
    6. 检查 If-None-Match 头，支持 304 响应
    7. 设置 Cache-Control 头（max-age=3600）
    8. 设置 Content-Type 为 application/vnd.apache.arrow.stream
  - **验收标准**：
    - 返回 Arrow IPC 二进制数据
    - ETag 机制正常工作
    - 304 响应正常返回
    - Cache-Control 头设置正确

- [ ]* 2.1 编写 Arrow 接口属性测试
  - **属性 1: Arrow 数据完整性**
  - **验证需求: REQ-A002**

- [ ]* 2.2 编写缓存机制属性测试
  - **属性 7: 缓存一致性**
  - **验证需求: REQ-C002**

- [x] 3. 实现 JSON 分析接口
  - **文件**: `features/business_analysis/routes.py`
  - **需求**: REQ-B003
  - **描述**: 创建 JSON 格式的聚合数据接口
  - **实施步骤**：
    1. 创建 `POST /customer-orders/analyze` 端点
    2. 定义请求模型 `AnalysisRequest`
       - group_by: Literal["customer_source", "group_type"]
       - time_grain: Optional[Literal["year", "month"]]
       - date_range: DateRange
    3. 定义响应模型 `AnalysisResponse`
       - data: List[AnalysisDataPoint]
       - summary: AnalysisSummary
       - execution_time: float
    4. 调用 Service 层 `analyze()` 方法
    5. 计算汇总信息
    6. 设置 HTTP 缓存头
    7. 返回 JSON 响应
  - **验收标准**：
    - 接口正常工作
    - 返回数据格式正确
    - 聚合计算准确
    - 缓存头设置正确

- [ ]* 3.1 编写 JSON 接口单元测试
  - 测试不同分组维度的聚合结果
  - 测试不同时间粒度的聚合结果
  - 测试金额格式化正确

- [x] 4. 实现错误处理
  - **文件**: `features/business_analysis/routes.py`
  - **需求**: REQ-C003
  - **描述**: 完善错误处理逻辑
  - **实施步骤**：
    1. 添加参数验证（使用 Pydantic）
       - 日期范围验证（结束日期不能早于开始日期）
       - 日期范围限制（不超过 5 年）
    2. 添加数据为空处理
       - 返回 404 状态码
       - 提示"所选时间范围内无数据"
    3. 添加数据库错误处理
       - 返回 500 状态码
       - 提示"系统繁忙，请稍后重试"
    4. 添加日志记录（使用 loguru）
  - **验收标准**：
    - 无效参数返回 422
    - 数据为空返回 404
    - 数据库错误返回 500
    - 错误信息友好

- [ ]* 4.1 编写错误处理属性测试
  - **属性 9: 错误响应完整性**
  - **验证需求: REQ-C003**

- [x] 5. 注册路由到主应用
  - **文件**: `main.py`
  - **需求**: REQ-A001, REQ-B001
  - **描述**: 将新接口注册到 FastAPI 主应用
  - **实施步骤**：
    1. 导入 business_analysis 路由
    2. 使用 `app.include_router()` 注册路由
    3. 验证路由可访问
  - **验收标准**：
    - 路由注册成功
    - 可以通过 Swagger 文档访问
    - 接口正常工作


## 前端任务

### 阶段 3：主页面和 Tab 切换

- [x] 6. 创建主页面组件
  - **文件**: `OneManage_web/src/views/business-analysis-v3/index.vue`
  - **需求**: REQ-A001, REQ-B001
  - **描述**: 创建主页面，实现 Tab 切换
  - **实施步骤**：
    1. 创建 `views/business-analysis-v3/` 目录
    2. 创建 `index.vue` 主页面
    3. 使用 Naive UI Tabs 组件
    4. 添加两个 Tab：
       - "业务分析"（默认）
       - "数据探索"
    5. 实现 Tab 切换逻辑
    6. 添加 Loading 状态
  - **验收标准**：
    - Tab 切换器正常显示
    - 默认显示"业务分析" Tab
    - 点击切换流畅，无闪烁

- [ ]* 6.1 编写 Tab 切换单元测试
  - 测试默认显示业务分析 Tab
  - 测试点击切换到数据探索 Tab
  - 测试 Tab 切换流畅

- [x] 7. 添加路由配置
  - **文件**: `OneManage_web/src/router/index.ts`
  - **需求**: REQ-A001, REQ-B001
  - **描述**: 添加新页面的路由配置
  - **实施步骤**：
    1. 在路由配置中添加新路由
       ```typescript
       {
         path: '/business-analysis-v3',
         name: 'BusinessAnalysisV3',
         component: () => import('@/views/business-analysis-v3/index.vue'),
         meta: {
           title: '业务分析（混合架构）',
           icon: 'chart-line'
         }
       }
       ```
    2. 在导航菜单中添加入口
    3. 保留旧版本路由（向后兼容）
  - **验收标准**：
    - 路由配置正确
    - 可以通过 URL 访问页面
    - 导航菜单显示正常

### 阶段 4：业务分析模型组件

- [x] 8. 创建配置面板组件
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/ConfigPanel.vue`
  - **需求**: REQ-B002
  - **描述**: 实现分析配置面板
  - **实施步骤**：
    1. 创建 ConfigPanel.vue 组件
    2. 使用 Naive UI Radio Group 实现分组维度选择
       - 客户来源
       - 客户分组
    3. 使用 Naive UI Radio Group 实现时间粒度选择
       - 不分组
       - 按年
       - 按月
    4. 使用 Naive UI Date Picker 实现时间范围选择
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

- [ ]* 8.1 编写配置面板单元测试
  - 测试所有配置项存在
  - 测试默认值正确
  - 测试配置变更触发分析

- [x] 9. 创建图表面板组件
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/ChartsPanel.vue`
  - **需求**: REQ-B004
  - **描述**: 使用 ECharts 实现图表展示
  - **实施步骤**：
    1. 创建 ChartsPanel.vue 组件
    2. 使用 Naive UI Tabs 实现图表切换
       - 柱状图 tab
       - 折线图 tab（仅在有时间粒度时显示）
    3. 实现柱状图
       - X 轴：维度值
       - Y 轴：订单数量和累计金额（双 Y 轴）
       - 支持 tooltip 交互
    4. 实现折线图
       - X 轴：时间周期
       - Y 轴：订单数量和累计金额（双 Y 轴）
       - 显示趋势变化
    5. 实现图表响应式
       - 监听数据变化自动更新
       - 监听窗口大小变化自动调整
  - **验收标准**：
    - 图表正常渲染
    - 数据展示正确
    - 交互流畅
    - 响应式布局正常

- [ ]* 9.1 编写图表面板属性测试
  - **属性 5: 图表数据映射正确性**
  - **验证需求: REQ-B004**

- [x] 10. 创建数据表格组件
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/DataTable.vue`
  - **需求**: REQ-B005
  - **描述**: 使用 Naive UI DataTable 实现数据表格展示
  - **实施步骤**：
    1. 创建 DataTable.vue 组件
    2. 配置 Naive UI DataTable
       - columns: 列定义（动态生成）
       - data: 数据源
       - pagination: 分页配置
       - sorter: 排序配置
    3. 实现动态列配置
       - 维度列（根据 groupBy 显示）
       - 时间周期列（根据 timeGrain 显示）
       - 订单数量列（数字类型）
       - 累计金额列（数字类型）
       - 平均金额列（数字类型）
    4. 实现数据格式化
       - 金额显示千分位分隔符
       - 保留 2 位小数
  - **验收标准**：
    - 表格正常渲染
    - 列配置正确
    - 支持排序
    - 金额格式化正确

- [ ]* 10.1 编写数据表格属性测试
  - **属性 4: 金额格式化一致性**
  - **验证需求: REQ-B005**
  - **属性 6: 表格列完整性**
  - **验证需求: REQ-B005**

- [x] 11. 实现业务分析 Composable
  - **文件**: `OneManage_web/src/views/business-analysis-v3/composables/useBusinessAnalysis.ts`
  - **需求**: REQ-B003
  - **描述**: 实现业务分析逻辑和状态管理
  - **实施步骤**：
    1. 创建 useBusinessAnalysis.ts
    2. 定义状态
       - config: 分析配置
       - data: 分析结果数据
       - loading: 加载状态
       - error: 错误信息
    3. 实现 API 调用函数
       - analyze(): 执行分析
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

- [x] 12. 组装业务分析模型组件
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/BusinessAnalysisModel.vue`
  - **需求**: REQ-B001 至 REQ-B005
  - **描述**: 组装所有子组件，实现完整的业务分析模型
  - **实施步骤**：
    1. 创建 BusinessAnalysisModel.vue 组件
    2. 导入所有子组件
       - ConfigPanel
       - ChartsPanel
       - DataTable
    3. 使用 useBusinessAnalysis composable
    4. 实现页面布局
       - 顶部：配置面板
       - 中部：图表面板
       - 底部：数据表格
    5. 实现组件间数据传递
       - 配置面板 → 分析逻辑
       - 分析结果 → 图表和表格
    6. 实现 Loading 状态展示
    7. 实现错误提示（使用 n-message）
  - **验收标准**：
    - 页面布局合理
    - 组件协作正常
    - 数据流转正确
    - 用户体验良好


- [x] 4. 实现分析模型选择器
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/ModelSelector.vue`
  - **需求**: REQ-B006
  - **描述**: 创建分析模型选择组件
  - **实施步骤**：
    1. 创建模型选择器组件
    2. 定义分析模型配置（基础聚合、RFM、复购、帕累托、波士顿矩阵）
    3. 实现模型切换逻辑
    4. 集成到业务分析 Tab 中
    5. 添加模型描述和优先级标识
  - **验收标准**：
    - 显示所有可用分析模型
    - 支持模型切换
    - 切换时更新配置面板
    - UI 美观，交互流畅

- [ ] 5. 实现 RFM 分析后端接口
  - **文件**: `OneManage/features/business_analysis/services/customer_order_service.py`, `OneManage/features/business_analysis/routes.py`
  - **需求**: REQ-B007
  - **描述**: 实现 RFM 客户价值分析算法和接口
  - **实施步骤**：
    1. 在 Service 层添加 `rfm_analysis()` 方法
    2. 使用 Polars 计算 R、F、M 指标
    3. 实现分位数分层算法
    4. 定义客户分层规则
    5. 创建 GET `/rfm-analysis` 接口
    6. 添加响应模型和验证
  - **验收标准**：
    - 正确计算 RFM 指标
    - 客户分层准确
    - 响应时间 < 2 秒
    - 返回完整的分层统计和明细

- [x] 6. 实现 RFM 分析前端组件
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/RFMAnalysis.vue`
  - **需求**: REQ-B008
  - **描述**: 创建 RFM 分析可视化组件
  - **实施步骤**：
    1. 创建 RFM 分析组件
    2. 实现指标卡片展示
    3. 配置饼图（客户分层分布）
    4. 配置柱状图（各层级贡献）
    5. 实现数据表格（分层明细）
    6. 添加交互功能（点击查看详情）
  - **验收标准**：
    - 图表美观，数据准确
    - 支持交互操作
    - 响应式布局
    - 加载状态和错误处理

- [x] 7. 实现复购分析后端接口
  - **文件**: `OneManage/features/business_analysis/services/customer_order_service.py`, `OneManage/features/business_analysis/routes.py`
  - **需求**: REQ-B009
  - **描述**: 实现客户复购分析算法和接口
  - **实施步骤**：
    1. 在 Service 层添加 `repurchase_analysis()` 方法
    2. 计算整体复购率
    3. 计算平均复购周期
    4. 识别流失预警客户
    5. 按客户来源统计复购指标
    6. 创建 GET `/repurchase-analysis` 接口
  - **验收标准**：
    - 复购率计算准确
    - 流失预警逻辑正确
    - 响应时间 < 2 秒
    - 支持按来源分组统计

- [x] 8. 实现复购分析前端组件
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/RepurchaseAnalysis.vue`
  - **需求**: REQ-B010
  - **描述**: 创建复购分析可视化组件
  - **实施步骤**：
    1. 创建复购分析组件
    2. 实现指标卡片（复购率、平均周期、流失预警数）
    3. 配置柱状图（不同来源复购率对比）
    4. 配置折线图（复购周期分布）
    5. 实现流失预警客户列表
    6. 添加导出功能
  - **验收标准**：
    - 指标展示清晰
    - 图表交互良好
    - 支持数据导出
    - 流失预警突出显示

- [x] 9. 实现帕累托分析后端接口
  - **文件**: `OneManage/features/business_analysis/services/customer_order_service.py`, `OneManage/features/business_analysis/routes.py`
  - **需求**: REQ-B011
  - **描述**: 实现帕累托分析（80/20 法则）算法和接口
  - **实施步骤**：
    1. 在 Service 层添加 `pareto_analysis()` 方法
    2. 按客户贡献金额排序
    3. 计算累计贡献占比
    4. 识别核心客户群体（贡献 80% 收入）
    5. 统计核心客户指标
    6. 创建 GET `/pareto-analysis` 接口
  - **验收标准**：
    - 帕累托计算准确
    - 80% 分界点正确
    - 核心客户识别准确
    - 响应时间 < 2 秒

- [x] 10. 实现帕累托分析前端组件
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/ParetoAnalysis.vue`
  - **需求**: REQ-B012
  - **描述**: 创建帕累托分析可视化组件
  - **实施步骤**：
    1. 创建帕累托分析组件
    2. 实现核心指标提示卡片
    3. 配置帕累托图（柱状图 + 折线图组合）
    4. 在 80% 位置添加分界线标注
    5. 实现核心客户列表表格
    6. 添加导出功能
  - **验收标准**：
    - 帕累托图清晰易读
    - 80% 分界线明显
    - 核心客户突出显示
    - 支持数据导出

- [x] 11. 实现波士顿矩阵后端接口
  - **文件**: `OneManage/features/business_analysis/services/customer_order_service.py`, `OneManage/features/business_analysis/routes.py`
  - **需求**: REQ-B013
  - **描述**: 实现波士顿矩阵客户组合分析算法和接口
  - **实施步骤**：
    1. 在 Service 层添加 `bcg_matrix_analysis()` 方法
    2. 计算客户增长率（近期 vs 历史）
    3. 计算客户贡献度（占总金额百分比）
    4. 使用中位数划分四个象限
    5. 客户象限分类（明星、金牛、问题、瘦狗）
    6. 创建 GET `/bcg-matrix` 接口
  - **验收标准**：
    - 增长率计算准确
    - 象限划分合理
    - 客户分类正确
    - 响应时间 < 2 秒

- [x] 12. 实现波士顿矩阵前端组件
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/BCGMatrix.vue`
  - **需求**: REQ-B014
  - **描述**: 创建波士顿矩阵可视化组件
  - **实施步骤**：
    1. 创建波士顿矩阵组件
    2. 实现四象限指标卡片
    3. 配置散点图（增长率 vs 贡献度）
    4. 用不同颜色区分象限
    5. 添加中位数分界线
    6. 实现象限统计表格
  - **验收标准**：
    - 散点图布局合理
    - 象限颜色区分明显
    - 支持点击查看客户详情
    - 分界线清晰可见

- [x] 13. 集成分析模型到主界面
  - **文件**: `OneManage_web/src/views/business-analysis-v3/BusinessAnalysisModel.vue`
  - **需求**: REQ-B006
  - **描述**: 将所有分析模型集成到业务分析主界面
  - **实施步骤**：
    1. 更新主界面布局
    2. 集成模型选择器
    3. 实现动态组件加载
    4. 添加模型间的状态管理
    5. 优化切换动画和加载状态
    6. 添加模型使用说明
  - **验收标准**：
    - 所有模型正常工作
    - 切换流畅无闪烁
    - 状态管理正确
    - 用户体验良好

### 阶段 5：数据探索组件

- [x] 14. 添加分析模型测试
  - **文件**: `OneManage/tests/test_business_analysis_models.py`, `OneManage_web/src/views/business-analysis-v3/__tests__/`
  - **需求**: 所有分析模型需求
  - **描述**: 为所有分析模型添加完整的测试覆盖
  - **实施步骤**：
    1. 创建后端分析模型测试
    2. 测试 RFM 分析算法
    3. 测试复购分析逻辑
    4. 测试帕累托分析计算
    5. 测试波士顿矩阵分类
    6. 添加前端组件测试
  - **验收标准**：
    - 所有分析算法测试通过
    - 边界情况处理正确
    - 前端组件渲染正常
    - 测试覆盖率 > 80%

- [x] 15. 实现 Perspective 数据加载 Composable
  - **文件**: `OneManage_web/src/views/business-analysis-v3/composables/usePerspectiveData.ts`
  - **需求**: REQ-A002
  - **描述**: 封装 Arrow 数据加载逻辑
  - **实施步骤**：
    1. 创建 usePerspectiveData composable
    2. 实现 loadArrowData() 方法
       - 动态导入 Perspective（懒加载）
       - 创建 Perspective Worker（单例模式）
       - 获取 Arrow 数据
       - 加载到 Perspective Table
    3. 实现错误处理和重试逻辑
    4. 添加加载状态管理
    5. 添加缓存状态显示（从响应头读取）
  - **验收标准**：
    - API 调用正常
    - 错误处理完善
    - 状态管理正确
    - 显示缓存命中状态

- [ ]* 15.1 编写 Perspective 数据加载属性测试
  - **属性 1: Arrow 数据完整性**
  - **验证需求: REQ-A002**

- [x] 16. 实现视图状态管理 Composable
  - **文件**: `OneManage_web/src/views/business-analysis-v3/composables/useViewState.ts`
  - **需求**: REQ-A004
  - **描述**: 实现视图配置的保存和恢复
  - **实施步骤**：
    1. 创建 useViewState composable
    2. 实现 saveView() 方法（保存到 localStorage）
    3. 实现 loadView() 方法
    4. 实现 listViews() 方法
    5. 实现 deleteView() 方法
    6. 实现视图配置验证
  - **验收标准**：
    - 视图可以保存和恢复
    - 视图列表正常显示
    - 可以删除视图
    - 配置验证正确

- [ ]* 16.1 编写视图状态属性测试
  - **属性 2: 视图配置往返一致性**
  - **验证需求: REQ-A004**

- [x] 17. 创建视图管理组件
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/ViewManager.vue`
  - **需求**: REQ-A004
  - **描述**: 实现视图管理侧边栏
  - **实施步骤**：
    1. 创建 ViewManager.vue 组件
    2. 显示已保存的视图列表
    3. 实现视图加载功能
    4. 实现视图删除功能
    5. 实现视图重命名功能
    6. 添加空状态提示
  - **验收标准**：
    - 视图列表正常显示
    - 可以加载视图
    - 可以删除视图
    - 可以重命名视图

- [x] 18. 创建数据探索组件
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/DataExplorer.vue`
  - **需求**: REQ-A001 至 REQ-A004
  - **描述**: 创建基于 Perspective 的数据探索组件
  - **实施步骤**：
    1. 创建 DataExplorer.vue 组件
    2. 添加工具栏
       - "加载数据"按钮
       - "保存视图"按钮
       - "导出数据"按钮
       - 日期范围选择器
    3. 集成 Perspective Viewer
       - 使用 perspective-viewer 组件
       - 配置默认视图
       - 监听视图变化事件
    4. 集成 ViewManager 组件
    5. 实现数据加载逻辑
    6. 实现视图保存逻辑
    7. 实现数据导出逻辑（CSV、JSON、Arrow）
    8. 添加 Loading 状态
    9. 添加错误处理
  - **验收标准**：
    - Perspective Viewer 正常渲染
    - 可以加载 Arrow 数据
    - 可以自由拖拽分析
    - 可以保存和恢复视图
    - 可以导出数据

- [ ]* 18.1 编写数据探索单元测试
  - 测试 Perspective Viewer 正确加载
  - 测试 Arrow 数据加载成功
  - 测试视图保存和恢复

### 阶段 6：UI/UX 重构

- [x] 19. 创建顶部工具栏组件
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/TopToolbar.vue`
  - **需求**: UI 重构
  - **描述**: 创建顶部数据加载和 Tab 切换工具栏
  - **实施步骤**：
    1. 创建 TopToolbar.vue 组件
    2. 实现日期范围选择器
       - 开始日期输入
       - 结束日期输入
       - 日期验证
    3. 添加"加载数据"按钮（绿色主题）
    4. 集成 Tab 切换（数据探索 / 业务分析）
    5. 添加全局操作按钮（导出、设置等）
    6. 实现响应式布局
  - **验收标准**：
    - 工具栏布局美观
    - 日期选择器正常工作
    - Tab 切换流畅
    - 响应式适配正常

- [x] 20. 重构主页面布局
  - **文件**: `OneManage_web/src/views/business-analysis-v3/index.vue`
  - **需求**: UI 重构
  - **描述**: 将主页面改为顶部工具栏 + 内容区域布局
  - **实施步骤**：
    1. 移除原有的 Tab 切换器
    2. 添加 TopToolbar 组件
    3. 调整内容区域布局
    4. 实现响应式布局
    5. 优化样式和间距
  - **验收标准**：
    - 布局结构清晰
    - 工具栏固定在顶部
    - 内容区域自适应高度
    - 响应式布局正常

- [x] 21. 重构业务分析模型为左右分栏布局
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/BusinessAnalysisModel.vue`
  - **需求**: UI 重构
  - **描述**: 将业务分析模型改为左侧配置 + 右侧结果的布局
  - **实施步骤**：
    1. 使用 CSS Grid 实现左右分栏
    2. 左侧：配置面板（固定宽度 300px）
    3. 右侧：图表和表格（自适应宽度）
    4. 添加分栏调整功能（可选）
    5. 实现响应式布局（平板/移动端）
    6. 优化样式和间距
  - **验收标准**：
    - 左右分栏布局正常
    - 左侧宽度固定 300px
    - 右侧自适应剩余空间
    - 响应式布局正常
    - 配置和结果同屏显示

- [x] 22. 重构配置面板为侧边栏样式
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/ConfigPanel.vue`
  - **需求**: UI 重构
  - **描述**: 将配置面板改为侧边栏样式，支持分组折叠
  - **实施步骤**：
    1. 调整布局为侧边栏样式
    2. 添加配置分组（基础配置、高级配置）
    3. 实现分组折叠功能
    4. 底部固定"开始分析"按钮
    5. 优化样式（减少间距，紧凑布局）
    6. 添加配置项图标
  - **验收标准**：
    - 侧边栏样式美观
    - 分组折叠功能正常
    - 底部按钮固定
    - 配置项清晰易读
    - 适配 300px 宽度

- [ ] 23. 优化图表和表格展示区域
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/ChartsPanel.vue`, `DataTable.vue`
  - **需求**: UI 重构
  - **描述**: 优化图表和表格在右侧区域的展示
  - **实施步骤**：
    1. 调整图表容器高度（充分利用空间）
    2. 优化表格布局（虚拟滚动）
    3. 添加全屏模式按钮
    4. 优化图表和表格的间距
    5. 添加快速切换视图功能
  - **验收标准**：
    - 图表充分利用空间
    - 表格虚拟滚动正常
    - 全屏模式正常工作
    - 布局美观，间距合理

- [ ] 24. 添加响应式布局支持
  - **文件**: 多个组件文件
  - **需求**: UI 重构
  - **描述**: 为所有重构的组件添加响应式布局支持
  - **实施步骤**：
    1. 桌面端（>1200px）：左右分栏
    2. 平板端（768px-1200px）：可折叠侧边栏
    3. 移动端（<768px）：垂直布局
    4. 添加媒体查询
    5. 测试不同屏幕尺寸
  - **验收标准**：
    - 桌面端布局正常
    - 平板端侧边栏可折叠
    - 移动端垂直布局正常
    - 所有断点过渡流畅

- [ ] 25. UI 重构验收测试
  - **文件**: 所有重构的组件
  - **需求**: UI 重构
  - **描述**: 全面测试 UI 重构后的功能和体验
  - **实施步骤**：
    1. 功能测试（所有功能正常）
    2. 布局测试（不同屏幕尺寸）
    3. 交互测试（流畅度、响应速度）
    4. 样式测试（一致性、美观度）
    5. 性能测试（加载速度、渲染性能）
    6. 可访问性测试（键盘导航、屏幕阅读器）
  - **验收标准**：
    - 所有功能正常工作
    - 布局在所有屏幕尺寸下正常
    - 交互流畅，无卡顿
    - 样式统一，美观
    - 性能达标
    - 可访问性良好

### 阶段 7：优化和完善（可选）

- [ ] 27. 实现性能监控
  - **文件**: `OneManage_web/src/views/business-analysis-v3/composables/usePerformanceMonitor.ts`
  - **需求**: REQ-C001
  - **描述**: 实现性能监控系统
  - **实施步骤**：
    1. 创建 usePerformanceMonitor composable
    2. 实现性能指标记录
       - 组件加载时间
       - 数据加载时间
       - 操作响应时间
    3. 实现性能报告生成
    4. 实现性能阈值检查
    5. 添加性能警告提示
  - **验收标准**：
    - 性能指标记录正确
    - 性能报告清晰
    - 阈值检查正常
    - 警告提示友好

- [ ] 28. 实现用户引导组件
  - **文件**: `OneManage_web/src/views/business-analysis-v3/components/UserGuide.vue`
  - **需求**: REQ-C004
  - **描述**: 实现用户引导功能
  - **实施步骤**：
    1. 创建 UserGuide.vue 组件
    2. 添加首次使用引导
       - 功能介绍
       - 操作说明
       - 场景选择建议
    3. 添加操作提示（Tooltip）
    4. 添加帮助文档链接
    5. 实现引导状态管理（localStorage）
  - **验收标准**：
    - 首次使用有引导
    - 引导内容清晰
    - 可以关闭引导
    - 帮助文档完整

- [ ] 29. 实现错误处理 Composable
  - **文件**: `OneManage_web/src/views/business-analysis-v3/composables/useErrorHandler.ts`
  - **需求**: REQ-C003
  - **描述**: 统一错误处理逻辑
  - **实施步骤**：
    1. 创建 useErrorHandler composable
    2. 实现错误分类
       - 网络错误
       - 数据错误
       - WebAssembly 错误
       - 未知错误
    3. 实现错误提示
       - 使用 Naive UI Message 组件
       - 提供友好的错误信息
       - 提供解决建议
    4. 实现错误日志记录
  - **验收标准**：
    - 错误分类正确
    - 错误提示友好
    - 解决建议明确
    - 日志记录完整

- [ ]* 29.1 编写错误处理属性测试
  - **属性 9: 错误响应完整性**
  - **验证需求: REQ-C003**

- [ ] 30. 优化前端性能
  - **文件**: 多个文件
  - **需求**: REQ-C001
  - **描述**: 优化前端性能
  - **实施步骤**：
    1. 实现 Perspective Worker 懒加载
       - 使用动态 import()
       - 单例模式避免重复创建
    2. 实现组件懒加载
       - 使用 defineAsyncComponent
       - 按需加载子组件
    3. 优化 WebAssembly 加载
       - 使用 Promise.all() 并行加载
       - 减少串行等待时间
    4. 实现数据缓存
       - 前端缓存策略
       - 避免重复请求
    5. 添加性能监控
       - 使用 Performance API
       - 记录关键指标
  - **验收标准**：
    - 首次加载时间 < 3 秒
    - 数据加载时间 < 2 秒（50W 行）
    - 操作响应时间 < 100ms
    - 性能监控正常


## 测试任务（可选）

- [ ]* 23. 后端集成测试
  - **文件**: `tests/test_business_analysis_hybrid_integration.py`
  - **需求**: REQ-A002, REQ-B003, REQ-C001
  - **描述**: 测试后端完整流程
  - **实施步骤**：
    1. 测试 Arrow 数据接口完整流程
       - 请求 → 数据库查询 → Arrow 序列化 → 响应
    2. 测试 JSON 分析接口完整流程
       - 请求 → 数据库查询 → 聚合计算 → JSON 响应
    3. 测试缓存机制
       - 首次请求 → 缓存 → 再次请求 → 304 响应
    4. 测试性能
       - 50W 行数据 < 2 秒
  - **验收标准**：
    - 所有测试通过
    - 性能测试达标

- [ ]* 24. 前端端到端测试
  - **文件**: `OneManage_web/src/views/business-analysis-v3/__tests__/e2e.test.ts`
  - **需求**: REQ-A001 至 REQ-C004
  - **描述**: 测试前端完整流程
  - **实施步骤**：
    1. 测试业务分析模型完整流程
       - 配置参数 → 开始分析 → 显示结果
    2. 测试数据探索完整流程
       - 加载数据 → 拖拽分析 → 保存视图
    3. 测试 Tab 切换
       - 业务分析 ↔ 数据探索
    4. 测试错误处理
       - 网络错误 → 错误提示
  - **验收标准**：
    - 所有测试通过
    - 覆盖率 > 70%

## 检查点

- [ ] 25. 检查点 1：后端接口完成
  - **描述**: 确保所有后端接口正常工作
  - **验收标准**：
    - Arrow 数据接口正常
    - JSON 分析接口正常
    - 缓存机制正常
    - 错误处理完善
    - 所有测试通过

- [ ] 26. 检查点 2：业务分析模型完成
  - **描述**: 确保业务分析模型功能完整
  - **验收标准**：
    - 配置面板正常
    - 图表展示正确
    - 数据表格正确
    - 用户体验良好

- [ ] 27. 检查点 3：数据探索完成
  - **描述**: 确保数据探索功能完整
  - **验收标准**：
    - Perspective 正常加载
    - 可以自由拖拽分析
    - 视图保存和恢复正常
    - 数据导出正常

- [ ] 28. 最终检查点：所有功能完成
  - **描述**: 确保所有功能正常工作，性能达标
  - **验收标准**：
    - 所有功能正常
    - 所有测试通过
    - 性能达标
    - 用户体验良好
    - 文档完整

---

## 任务依赖关系

```
后端任务流程：
任务1 → 任务2 → 任务3 → 任务4 → 任务5 → 检查点1

前端任务流程：
任务6 → 任务7
任务6 → 任务8 → 任务9 → 任务10 → 任务11 → 任务12 → 检查点2
任务6 → 任务13 → 任务14 → 任务15 → 任务16 → 检查点3

优化任务：
检查点2 + 检查点3 → 任务17 → 任务18 → 任务19 → 任务20

测试任务（可选）：
检查点1 → 任务21
检查点2 + 检查点3 → 任务22

最终检查：
任务20 → 检查点4
```

## 任务优先级

**P0（核心功能）**：
- 任务 1-5（后端接口）
- 任务 6-7（主页面和路由）
- 任务 8-12（业务分析模型）
- 任务 13-16（数据探索）

**P1（增强功能）**：
- 任务 17-20（优化和完善）

**P2（可选功能）**：
- 任务 21-22（测试）

## 预估工作量

- 后端任务（任务 1-5）：2-3 天
- 前端主页面（任务 6-7）：0.5 天
- 业务分析模型（任务 8-12）：2-3 天
- 数据探索（任务 13-16）：2-3 天
- 优化和完善（任务 17-20）：1-2 天
- 测试任务（任务 21-22）：1 天（可选）

**总计**：8-12 天（不含测试）

## 验收标准总览

### 功能验收
- ✅ 用户可以在两个场景之间切换
- ✅ 业务分析模型：可以配置参数并查看固定图表
- ✅ 数据探索：可以自由拖拽分析数据
- ✅ 视图可以保存和恢复
- ✅ 数据可以导出
- ✅ 错误情况有友好提示

### 性能验收
- ✅ Arrow 数据加载 < 2 秒（50W 行）
- ✅ JSON 数据加载 < 2 秒（50W 行）
- ✅ 前端操作响应 < 100ms
- ✅ 图表渲染 < 500ms
- ✅ 缓存命中响应 < 50ms

### 代码质量验收
- ✅ 代码结构清晰，符合项目规范
- ✅ 关键逻辑有中文注释
- ✅ 错误处理完善
- ✅ 性能监控完整

## 注意事项

1. **遵循项目规范**：
   - 所有 Python 命令使用 `pdm run` 前缀
   - 代码注释使用简体中文
   - 遵循 Spec 任务范围规范

2. **性能优化**：
   - 后端使用 ConnectorX 加速数据加载
   - 前端使用懒加载和虚拟滚动
   - 使用 HTTP 缓存减少重复请求

3. **错误处理**：
   - 后端统一错误响应格式
   - 前端友好的错误提示
   - 日志记录关键操作

4. **扩展性**：
   - 代码设计考虑未来扩展
   - 两个场景独立，易于维护
   - 组件可复用

5. **测试策略**：
   - 单元测试覆盖核心逻辑
   - 属性测试验证通用属性
   - 集成测试验证完整流程

6. **渐进式迁移**：
   - 保留旧版本功能
   - 新旧版本并存
   - 逐步引导用户迁移

## 技术栈

### 后端
- FastAPI 0.100+
- Polars 0.20+
- PyArrow 20.0+
- SQLAlchemy 2.0+
- Pydantic 2.0+
- ConnectorX 0.4+

### 前端
- Vue 3.3+
- Naive UI 2.35+
- Perspective 4.0+
- ECharts 6.0+
- TypeScript 5.0+
- Vite 5.0+

### 测试
- Pytest（后端）
- Hypothesis（后端属性测试）
- Vitest（前端）
- fast-check（前端属性测试）

## 文档

- 需求文档：`.kiro/specs/business-analysis-hybrid/requirements.md`
- 设计文档：`.kiro/specs/business-analysis-hybrid/design.md`
- 任务清单：`.kiro/specs/business-analysis-hybrid/tasks.md`（本文档）
- API 文档：Swagger UI（自动生成）
- 用户手册：待补充

## 参考资料

- Perspective 文档：https://perspective.finos.org/
- ECharts 文档：https://echarts.apache.org/
- Polars 文档：https://pola-rs.github.io/polars/
- Arrow 文档：https://arrow.apache.org/
- Naive UI 文档：https://www.naiveui.com/
