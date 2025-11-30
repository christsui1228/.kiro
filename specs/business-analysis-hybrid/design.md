# 业务分析混合架构设计文档

## 概述

本文档描述基于混合架构的业务分析功能设计，将数据探索和业务分析模型分离为两个独立的功能模块。

### 核心设计理念

**两个独立的功能，两种不同的数据流**：

1. **场景 A：数据探索** - 使用 Perspective 实现自由拖拽分析
2. **场景 B：业务分析模型** - 使用 ECharts 实现固定图表展示

### 设计目标

- 清晰分离两个场景，避免架构混乱
- 各自使用最适合的技术栈
- 保持代码简洁，避免不必要的复杂度
- 提供优秀的用户体验

## 架构设计

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                      前端 Vue 3 应用                          │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────┐         ┌─────────────────┐            │
│  │  场景 A：数据探索 │         │ 场景 B：业务分析  │            │
│  │  (Perspective)  │         │   (ECharts)     │            │
│  └─────────────────┘         └─────────────────┘            │
│         │                            │                       │
│         │ Arrow IPC                  │ JSON                  │
│         ↓                            ↓                       │
└─────────────────────────────────────────────────────────────┘
         │                            │
         ↓                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      后端 FastAPI 应用                        │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────┐         ┌─────────────────┐            │
│  │  Arrow 数据接口  │         │  JSON 分析接口   │            │
│  │  (原始数据)      │         │  (聚合数据)      │            │
│  └─────────────────┘         └─────────────────┘            │
│         │                            │                       │
│         └────────────┬───────────────┘                       │
│                      ↓                                       │
│              ┌──────────────┐                                │
│              │ Polars 引擎   │                                │
│              └──────────────┘                                │
│                      ↓                                       │
└─────────────────────────────────────────────────────────────┘
                       │
                       ↓
              ┌──────────────┐
              │  PostgreSQL  │
              └──────────────┘
```


### 数据流对比

#### 场景 A：数据探索（Perspective）

```
用户操作
    ↓
前端 Perspective Viewer
    ↓
加载 Arrow 数据（首次）
    ↓
GET /api/business-analysis/customer-orders/arrow
    ↓
后端 Service 层
    ↓
Polars 查询原始数据（JOIN customers + orders）
    ↓
添加时间维度列（year, month, quarter）
    ↓
Arrow IPC 序列化
    ↓
HTTP 响应（Content-Type: application/vnd.apache.arrow.stream）
    ↓
前端 Perspective 加载数据
    ↓
用户自由拖拽分析（无需后端参与）
    ↓
Perspective 内部计算聚合（WebAssembly 加速）
```

#### 场景 B：业务分析模型（ECharts）

```
用户配置参数（维度、时间范围）
    ↓
点击"开始分析"
    ↓
POST /api/business-analysis/customer-orders/analyze
    ↓
后端 Service 层
    ↓
Polars 查询原始数据（JOIN customers + orders）
    ↓
Polars 聚合计算（group_by + agg）
    ↓
格式化结果（保留 2 位小数）
    ↓
JSON 序列化
    ↓
HTTP 响应（Content-Type: application/json）
    ↓
前端接收 JSON 数据
    ↓
格式化为 ECharts 配置
    ↓
ECharts 渲染图表
```

## 组件设计

### 后端组件

#### 1. Arrow 数据接口

**文件**: `features/business_analysis/routes.py`

**接口定义**:
```python
@router.get("/customer-orders/arrow")
async def get_customer_orders_arrow(
    date_range: DateRange = Query(...),
    db: AsyncSession = Depends(get_db)
) -> Response:
    """
    返回 Arrow IPC 格式的原始数据
    用于前端 Perspective 自由分析
    """
```

**功能**:
- 查询原始数据（不做聚合）
- 添加时间维度列（year, month, quarter, day_of_week）
- 转换为 Arrow IPC 格式
- 设置 HTTP 缓存头（ETag + Cache-Control）

**返回数据结构**:
```
customer_global_id: string
customer_source: string
group_type: string
customer_nickname: string
order_id: string
order_amount: float
order_created_date: date
year: int
month: int
quarter: int
day_of_week: int
```


#### 2. JSON 分析接口

**文件**: `features/business_analysis/routes.py`

**接口定义**:
```python
@router.post("/customer-orders/analyze")
async def analyze_customer_orders(
    request: AnalysisRequest,
    db: AsyncSession = Depends(get_db)
) -> AnalysisResponse:
    """
    返回 JSON 格式的聚合数据
    用于前端 ECharts 固定图表展示
    """
```

**请求参数**:
```python
class AnalysisRequest(BaseModel):
    group_by: Literal["customer_source", "group_type"]
    time_grain: Optional[Literal["year", "month"]] = None
    date_range: DateRange
```

**响应数据**:
```python
class AnalysisResponse(BaseModel):
    data: List[AnalysisDataPoint]
    summary: AnalysisSummary
    execution_time: float

class AnalysisDataPoint(BaseModel):
    dimension_value: str
    time_period: Optional[str] = None
    order_count: int
    total_amount: float
    avg_amount: float
```

**功能**:
- 查询原始数据
- 根据参数进行聚合计算
- 格式化金额（保留 2 位小数）
- 计算汇总信息
- 设置 HTTP 缓存头

#### 3. Service 层

**文件**: `features/business_analysis/services/customer_order_service.py`

**类设计**:
```python
class CustomerOrderService:
    """客户订单分析服务"""
    
    async def get_raw_data(
        self,
        date_range: DateRange,
        db: AsyncSession
    ) -> pl.DataFrame:
        """
        获取原始数据（用于 Perspective）
        返回明细数据，不做聚合
        """
    
    async def analyze(
        self,
        request: AnalysisRequest,
        db: AsyncSession
    ) -> pl.DataFrame:
        """
        执行聚合分析（用于 ECharts）
        返回聚合后的数据
        """
```

**关键方法**:

1. `get_raw_data()` - 获取原始数据
   - 构建 SQL 查询（JOIN customers + orders）
   - 添加时间维度列
   - 返回 Polars DataFrame

2. `analyze()` - 执行聚合分析
   - 调用 `get_raw_data()` 获取原始数据
   - 根据 `group_by` 和 `time_grain` 进行聚合
   - 计算订单数量、累计金额、平均金额
   - 格式化结果

#### 4. 高级分析模型服务

**文件**: `features/business_analysis/services/analysis_models.py`

这是新增的服务文件，包含各种高级分析模型的实现。

**类设计**:
```python
class RFMAnalysisService:
    """RFM 客户价值分析服务"""
    
    async def analyze_rfm(
        self,
        db: AsyncSession,
        date_range: DateRange,
        reference_date: date
    ) -> pl.DataFrame:
        """
        执行 RFM 分析
        
        计算逻辑：
        1. R (Recency): reference_date - 最后购买日期
        2. F (Frequency): 购买次数
        3. M (Monetary): 总购买金额
        4. 使用分位数方法分为 5 个等级
        5. 根据 RFM 组合分为 8 个客户层级
        
        Returns:
            包含客户分层结果的 DataFrame
        """

class RepurchaseAnalysisService:
    """客户复购分析服务"""
    
    async def analyze_repurchase(
        self,
        db: AsyncSession,
        date_range: DateRange
    ) -> dict:
        """
        执行复购分析
        
        计算逻辑：
        1. 整体复购率 = 有复购的客户数 / 总客户数
        2. 平均复购周期 = 两次购买间隔的平均值
        3. 流失预警 = 超过平均周期 1.5 倍未购买的客户
        4. 按客户来源/分组统计复购指标
        
        Returns:
            包含复购指标和流失预警客户的字典
        """

class ParetoAnalysisService:
    """帕累托分析服务（80/20 法则）"""
    
    async def analyze_pareto(
        self,
        db: AsyncSession,
        date_range: DateRange
    ) -> pl.DataFrame:
        """
        执行帕累托分析
        
        计算逻辑：
        1. 按客户贡献金额降序排序
        2. 计算累计金额和累计占比
        3. 识别贡献 80% 金额的客户群体
        4. 统计核心客户数量和占比
        
        Returns:
            包含客户排名和累计占比的 DataFrame
        """

class BCGMatrixService:
    """波士顿矩阵分析服务"""
    
    async def analyze_bcg_matrix(
        self,
        db: AsyncSession,
        date_range: DateRange,
        comparison_period_days: int = 90
    ) -> pl.DataFrame:
        """
        执行波士顿矩阵分析
        
        计算逻辑：
        1. 将时间范围分为两个周期（近期 vs 历史）
        2. 计算增长率 = (近期金额 - 历史金额) / 历史金额
        3. 计算贡献度 = 客户金额 / 总金额
        4. 根据增长率和贡献度的中位数分为四个象限
        
        Returns:
            包含客户象限分类的 DataFrame
        """
```

**API 接口设计**:

```python
# RFM 分析接口
@router.post("/rfm-analysis")
async def rfm_analysis(
    request: RFMAnalysisRequest,
    db: AsyncSession = Depends(get_db)
) -> RFMAnalysisResponse:
    """RFM 客户价值分析"""

# 复购分析接口
@router.post("/repurchase-analysis")
async def repurchase_analysis(
    request: RepurchaseAnalysisRequest,
    db: AsyncSession = Depends(get_db)
) -> RepurchaseAnalysisResponse:
    """客户复购分析"""

# 帕累托分析接口
@router.post("/pareto-analysis")
async def pareto_analysis(
    request: ParetoAnalysisRequest,
    db: AsyncSession = Depends(get_db)
) -> ParetoAnalysisResponse:
    """帕累托分析（80/20 法则）"""

# 波士顿矩阵接口
@router.post("/bcg-matrix")
async def bcg_matrix_analysis(
    request: BCGMatrixRequest,
    db: AsyncSession = Depends(get_db)
) -> BCGMatrixResponse:
    """波士顿矩阵分析"""
```


### 前端组件

#### 1. 主页面组件

**文件**: `OneManage_web/src/views/business-analysis-v3/index.vue`

**组件结构**:
```vue
<template>
  <div class="business-analysis-v3">
    <!-- Tab 切换器 -->
    <n-tabs v-model:value="activeTab" type="segment">
      <n-tab-pane name="model" tab="业务分析">
        <BusinessAnalysisModel />
      </n-tab-pane>
      <n-tab-pane name="explore" tab="数据探索">
        <DataExplorer />
      </n-tab-pane>
    </n-tabs>
  </div>
</template>
```

**功能**:
- 提供 Tab 切换
- 懒加载子组件
- 管理全局状态

#### 2. 业务分析模型组件

**文件**: `OneManage_web/src/views/business-analysis-v3/components/BusinessAnalysisModel.vue`

**组件结构**:
```vue
<template>
  <div class="business-analysis-model">
    <!-- 配置面板 -->
    <ConfigPanel @analyze="handleAnalyze" />
    
    <!-- 图表展示 -->
    <n-spin :show="loading">
      <ChartsPanel :data="analysisData" />
    </n-spin>
    
    <!-- 数据表格 -->
    <DataTable :data="analysisData" />
  </div>
</template>
```

**子组件**:

1. **ConfigPanel** - 配置面板
   - 分组维度选择（Radio Group）
   - 时间粒度选择（Radio Group）
   - 时间范围选择（Date Picker）
   - 开始分析按钮

2. **ChartsPanel** - 图表面板
   - 柱状图（订单数量和金额对比）
   - 折线图（时间趋势）
   - 使用 ECharts 6.0

3. **DataTable** - 数据表格
   - 使用 Naive UI DataTable
   - 支持排序
   - 金额格式化

**Composable**:
```typescript
// composables/useBusinessAnalysis.ts
export function useBusinessAnalysis() {
  const loading = ref(false)
  const analysisData = ref<AnalysisData[]>([])
  
  const analyze = async (params: AnalysisParams) => {
    loading.value = true
    try {
      const response = await fetch('/api/business-analysis/customer-orders/analyze', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(params)
      })
      const data = await response.json()
      analysisData.value = data.data
    } finally {
      loading.value = false
    }
  }
  
  return { loading, analysisData, analyze }
}
```


#### 3. 数据探索组件

**文件**: `OneManage_web/src/views/business-analysis-v3/components/DataExplorer.vue`

**组件结构**:
```vue
<template>
  <div class="data-explorer">
    <!-- 工具栏 -->
    <div class="toolbar">
      <n-button @click="loadData" :loading="loading">
        加载数据
      </n-button>
      <n-button @click="saveView">保存视图</n-button>
      <n-button @click="exportData">导出数据</n-button>
    </div>
    
    <!-- Perspective Viewer -->
    <perspective-viewer
      ref="viewerRef"
      class="perspective-viewer"
    />
    
    <!-- 视图管理侧边栏 -->
    <ViewManager @load-view="loadView" />
  </div>
</template>
```

**功能**:
- 加载 Arrow 数据
- 集成 Perspective Viewer
- 视图保存和恢复
- 数据导出

**Composable**:
```typescript
// composables/usePerspectiveData.ts
export function usePerspectiveData() {
  const loading = ref(false)
  const table = ref<Table | null>(null)
  
  const loadArrowData = async (dateRange: DateRange) => {
    loading.value = true
    try {
      // 加载 Perspective Worker
      const perspective = await import('@finos/perspective')
      const worker = perspective.default.worker()
      
      // 获取 Arrow 数据
      const response = await fetch(
        `/api/business-analysis/customer-orders/arrow?start=${dateRange.start}&end=${dateRange.end}`
      )
      const arrayBuffer = await response.arrayBuffer()
      
      // 加载到 Perspective
      table.value = await worker.table(arrayBuffer)
      
      return table.value
    } finally {
      loading.value = false
    }
  }
  
  return { loading, table, loadArrowData }
}
```

**视图管理**:
```typescript
// composables/useViewState.ts
export function useViewState() {
  const savedViews = ref<SavedView[]>([])
  
  const saveView = (name: string, config: ViewConfig) => {
    const view: SavedView = {
      id: Date.now().toString(),
      name,
      config,
      createdAt: new Date()
    }
    savedViews.value.push(view)
    localStorage.setItem('perspective-views', JSON.stringify(savedViews.value))
  }
  
  const loadView = (id: string) => {
    const view = savedViews.value.find(v => v.id === id)
    return view?.config
  }
  
  const deleteView = (id: string) => {
    savedViews.value = savedViews.value.filter(v => v.id !== id)
    localStorage.setItem('perspective-views', JSON.stringify(savedViews.value))
  }
  
  return { savedViews, saveView, loadView, deleteView }
}
```


## 数据模型

### 后端数据模型

#### 1. Arrow 数据模型（原始数据）

```python
# Polars DataFrame Schema
{
    "customer_global_id": pl.Utf8,
    "customer_source": pl.Utf8,
    "group_type": pl.Utf8,
    "customer_nickname": pl.Utf8,
    "order_id": pl.Utf8,
    "order_amount": pl.Float64,
    "order_created_date": pl.Date,
    "year": pl.Int32,
    "month": pl.Int32,
    "quarter": pl.Int32,
    "day_of_week": pl.Int32
}
```

#### 2. JSON 分析数据模型（聚合数据）

```python
class DateRange(BaseModel):
    start: date
    end: date

class AnalysisRequest(BaseModel):
    group_by: Literal["customer_source", "group_type"]
    time_grain: Optional[Literal["year", "month"]] = None
    date_range: DateRange

class AnalysisDataPoint(BaseModel):
    dimension_value: str
    time_period: Optional[str] = None
    order_count: int
    total_amount: float
    avg_amount: float

class AnalysisSummary(BaseModel):
    total_orders: int
    total_amount: float
    avg_amount: float
    date_range: DateRange

class AnalysisResponse(BaseModel):
    data: List[AnalysisDataPoint]
    summary: AnalysisSummary
    execution_time: float
```

### 前端数据模型

#### 1. 业务分析模型

```typescript
// 分析参数
interface AnalysisParams {
  groupBy: 'customer_source' | 'group_type'
  timeGrain?: 'year' | 'month'
  dateRange: {
    start: string
    end: string
  }
}

// 分析数据点
interface AnalysisDataPoint {
  dimensionValue: string
  timePeriod?: string
  orderCount: number
  totalAmount: number
  avgAmount: number
}

// 分析响应
interface AnalysisResponse {
  data: AnalysisDataPoint[]
  summary: {
    totalOrders: number
    totalAmount: number
    avgAmount: number
    dateRange: {
      start: string
      end: string
    }
  }
  executionTime: number
}
```

#### 2. 数据探索模型

```typescript
// 视图配置
interface ViewConfig {
  columns: string[]
  group_by: string[]
  split_by: string[]
  aggregates: Record<string, string>
  filter: any[]
  sort: any[]
  plugin: string
}

// 保存的视图
interface SavedView {
  id: string
  name: string
  config: ViewConfig
  createdAt: Date
}

// 日期范围
interface DateRange {
  start: string
  end: string
}
```


## 正确性属性

*一个属性是一个特征或行为，应该在系统的所有有效执行中保持为真——本质上是关于系统应该做什么的正式陈述。属性作为人类可读规范和机器可验证正确性保证之间的桥梁。*

### 数据加载属性

**属性 1: Arrow 数据完整性**
*对于任意*有效的日期范围，调用 Arrow 数据接口应该返回包含所有必要列的数据，且数据格式为 Arrow IPC
**验证需求: REQ-A002**

**属性 2: 视图配置往返一致性**
*对于任意*视图配置，保存后再加载应该得到相同的配置对象
**验证需求: REQ-A004**

### 聚合计算属性

**属性 3: 聚合结果正确性**
*对于任意*有效的分析参数（分组维度、时间粒度、日期范围），后端返回的聚合结果应该满足：
- 订单数量 = 去重后的 order_id 数量
- 累计金额 = order_amount 的总和
- 平均金额 = 累计金额 / 订单数量
**验证需求: REQ-B003**

**属性 4: 金额格式化一致性**
*对于任意*金额数值，格式化后应该保留 2 位小数且包含千分位分隔符
**验证需求: REQ-B005**

### 图表渲染属性

**属性 5: 图表数据映射正确性**
*对于任意*聚合数据，转换为 ECharts 配置后，图表的数据点数量应该等于聚合数据的行数
**验证需求: REQ-B004**

**属性 6: 表格列完整性**
*对于任意*聚合数据，渲染的表格应该包含所有必要的列（维度值、时间周期、订单数量、累计金额、平均金额）
**验证需求: REQ-B005**

### 缓存属性

**属性 7: 缓存一致性**
*对于任意*请求，如果 ETag 匹配，服务器应该返回 304 状态码且不返回数据体
**验证需求: REQ-C002**

**属性 8: 缓存数据等价性**
*对于任意*请求，缓存命中时返回的数据应该与首次请求返回的数据完全相同
**验证需求: REQ-C002**

### 错误处理属性

**属性 9: 错误响应完整性**
*对于任意*错误场景（网络错误、数据为空、服务器错误），系统应该返回包含错误消息和解决建议的响应
**验证需求: REQ-C003**


## 错误处理

### 后端错误处理

#### 1. 参数验证错误

**场景**: 用户提供无效的参数

**处理**:
```python
# 使用 Pydantic 自动验证
class AnalysisRequest(BaseModel):
    group_by: Literal["customer_source", "group_type"]
    time_grain: Optional[Literal["year", "month"]] = None
    date_range: DateRange
    
    @validator('date_range')
    def validate_date_range(cls, v):
        if v.end < v.start:
            raise ValueError("结束日期不能早于开始日期")
        if (v.end - v.start).days > 1825:  # 5 年
            raise ValueError("日期范围不能超过 5 年")
        return v
```

**响应**:
```json
{
  "detail": "结束日期不能早于开始日期"
}
```

**HTTP 状态码**: 422 Unprocessable Entity

#### 2. 数据为空错误

**场景**: 查询结果为空

**处理**:
```python
if df.height == 0:
    raise HTTPException(
        status_code=404,
        detail="所选时间范围内无数据，请调整查询条件"
    )
```

**响应**:
```json
{
  "detail": "所选时间范围内无数据，请调整查询条件"
}
```

**HTTP 状态码**: 404 Not Found

#### 3. 数据库错误

**场景**: 数据库连接失败或查询错误

**处理**:
```python
try:
    result = await db.execute(query)
except Exception as e:
    logger.error(f"数据库查询失败: {e}")
    raise HTTPException(
        status_code=500,
        detail="系统繁忙，请稍后重试"
    )
```

**响应**:
```json
{
  "detail": "系统繁忙，请稍后重试"
}
```

**HTTP 状态码**: 500 Internal Server Error

### 前端错误处理

#### 1. 网络错误

**场景**: API 请求失败

**处理**:
```typescript
try {
  const response = await fetch(url)
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`)
  }
} catch (error) {
  message.error('网络连接失败，请检查网络后重试')
  console.error('API 请求失败:', error)
}
```

#### 2. WebAssembly 加载失败

**场景**: Perspective WebAssembly 加载失败

**处理**:
```typescript
try {
  const perspective = await import('@finos/perspective')
} catch (error) {
  message.error('浏览器不支持 WebAssembly，请使用 Chrome 90+ 或 Firefox 88+')
  console.error('Perspective 加载失败:', error)
}
```

#### 3. 数据格式错误

**场景**: 后端返回的数据格式不符合预期

**处理**:
```typescript
try {
  const data = await response.json()
  if (!Array.isArray(data.data)) {
    throw new Error('数据格式错误')
  }
} catch (error) {
  message.error('数据格式错误，请联系管理员')
  console.error('数据解析失败:', error)
}
```


## 测试策略

### 单元测试

#### 后端单元测试

**测试文件**: `tests/test_business_analysis_hybrid.py`

**测试用例**:

1. **测试 Arrow 数据接口**
   - 测试返回的数据格式为 Arrow IPC
   - 测试数据包含所有必要的列
   - 测试 ETag 生成正确
   - 测试 Cache-Control 头设置正确

2. **测试 JSON 分析接口**
   - 测试不同分组维度的聚合结果
   - 测试不同时间粒度的聚合结果
   - 测试金额格式化正确（2 位小数）
   - 测试汇总信息计算正确

3. **测试 Service 层**
   - 测试 `get_raw_data()` 返回原始数据
   - 测试 `analyze()` 返回聚合数据
   - 测试时间维度列添加正确

4. **测试错误处理**
   - 测试无效参数返回 422
   - 测试数据为空返回 404
   - 测试数据库错误返回 500

#### 前端单元测试

**测试文件**: `OneManage_web/src/views/business-analysis-v3/__tests__/`

**测试用例**:

1. **测试 Tab 切换**
   - 测试默认显示业务分析 Tab
   - 测试点击切换到数据探索 Tab
   - 测试 Tab 切换流畅

2. **测试配置面板**
   - 测试所有配置项存在
   - 测试默认值正确
   - 测试配置变更触发分析

3. **测试图表渲染**
   - 测试 ECharts 正确渲染
   - 测试图表数据映射正确
   - 测试图表配置正确

4. **测试表格渲染**
   - 测试表格包含所有列
   - 测试金额格式化正确
   - 测试排序功能正常

5. **测试 Perspective 集成**
   - 测试 Perspective Viewer 正确加载
   - 测试 Arrow 数据加载成功
   - 测试视图保存和恢复

### 属性测试

使用 Hypothesis（Python）和 fast-check（TypeScript）进行属性测试。

#### 后端属性测试

**测试文件**: `tests/test_business_analysis_properties.py`

**测试用例**:

1. **属性 1: Arrow 数据完整性**
```python
from hypothesis import given, strategies as st

@given(
    start_date=st.dates(min_value=date(2020, 1, 1), max_value=date(2024, 12, 31)),
    end_date=st.dates(min_value=date(2020, 1, 1), max_value=date(2024, 12, 31))
)
async def test_arrow_data_completeness(start_date, end_date):
    """属性 1: Arrow 数据完整性"""
    if end_date < start_date:
        start_date, end_date = end_date, start_date
    
    response = await client.get(
        f"/api/business-analysis/customer-orders/arrow?start={start_date}&end={end_date}"
    )
    
    assert response.status_code == 200
    assert response.headers["content-type"] == "application/vnd.apache.arrow.stream"
    
    # 验证数据包含所有必要的列
    table = pa.ipc.open_stream(response.content).read_all()
    required_columns = [
        "customer_global_id", "customer_source", "group_type",
        "order_id", "order_amount", "order_created_date",
        "year", "month", "quarter", "day_of_week"
    ]
    assert all(col in table.column_names for col in required_columns)
```

2. **属性 3: 聚合结果正确性**
```python
@given(
    group_by=st.sampled_from(["customer_source", "group_type"]),
    time_grain=st.one_of(st.none(), st.sampled_from(["year", "month"])),
    start_date=st.dates(min_value=date(2020, 1, 1), max_value=date(2024, 12, 31)),
    end_date=st.dates(min_value=date(2020, 1, 1), max_value=date(2024, 12, 31))
)
async def test_aggregation_correctness(group_by, time_grain, start_date, end_date):
    """属性 3: 聚合结果正确性"""
    if end_date < start_date:
        start_date, end_date = end_date, start_date
    
    response = await client.post(
        "/api/business-analysis/customer-orders/analyze",
        json={
            "group_by": group_by,
            "time_grain": time_grain,
            "date_range": {"start": str(start_date), "end": str(end_date)}
        }
    )
    
    assert response.status_code == 200
    data = response.json()
    
    # 验证聚合结果正确性
    for point in data["data"]:
        assert point["order_count"] > 0
        assert point["total_amount"] > 0
        assert abs(point["avg_amount"] - point["total_amount"] / point["order_count"]) < 0.01
```

3. **属性 7: 缓存一致性**
```python
@given(
    start_date=st.dates(min_value=date(2020, 1, 1), max_value=date(2024, 12, 31)),
    end_date=st.dates(min_value=date(2020, 1, 1), max_value=date(2024, 12, 31))
)
async def test_cache_consistency(start_date, end_date):
    """属性 7: 缓存一致性"""
    if end_date < start_date:
        start_date, end_date = end_date, start_date
    
    # 第一次请求
    response1 = await client.get(
        f"/api/business-analysis/customer-orders/arrow?start={start_date}&end={end_date}"
    )
    assert response1.status_code == 200
    etag = response1.headers["etag"]
    
    # 第二次请求，带 If-None-Match
    response2 = await client.get(
        f"/api/business-analysis/customer-orders/arrow?start={start_date}&end={end_date}",
        headers={"If-None-Match": etag}
    )
    assert response2.status_code == 304
    assert len(response2.content) == 0
```

#### 前端属性测试

**测试文件**: `OneManage_web/src/views/business-analysis-v3/__tests__/properties.test.ts`

**测试用例**:

1. **属性 2: 视图配置往返一致性**
```typescript
import fc from 'fast-check'

test('属性 2: 视图配置往返一致性', () => {
  fc.assert(
    fc.property(
      fc.record({
        columns: fc.array(fc.string()),
        group_by: fc.array(fc.string()),
        aggregates: fc.dictionary(fc.string(), fc.string())
      }),
      (config) => {
        // 保存视图
        const { saveView, loadView } = useViewState()
        const viewId = 'test-view'
        saveView(viewId, config)
        
        // 加载视图
        const loadedConfig = loadView(viewId)
        
        // 验证一致性
        expect(loadedConfig).toEqual(config)
      }
    ),
    { numRuns: 100 }
  )
})
```

2. **属性 4: 金额格式化一致性**
```typescript
test('属性 4: 金额格式化一致性', () => {
  fc.assert(
    fc.property(
      fc.float({ min: 0, max: 1000000 }),
      (amount) => {
        const formatted = formatAmount(amount)
        
        // 验证格式
        expect(formatted).toMatch(/^\d{1,3}(,\d{3})*\.\d{2}$/)
        
        // 验证精度
        const parsed = parseFloat(formatted.replace(/,/g, ''))
        expect(Math.abs(parsed - amount)).toBeLessThan(0.01)
      }
    ),
    { numRuns: 100 }
  )
})
```

### 集成测试

**测试场景**:

1. **端到端数据流测试**
   - 测试从数据库查询到前端渲染的完整流程
   - 测试数据探索场景的完整流程
   - 测试业务分析场景的完整流程

2. **缓存机制测试**
   - 测试 HTTP 缓存正确工作
   - 测试缓存失效机制
   - 测试缓存命中率

3. **性能测试**
   - 测试 50W 行数据的加载时间
   - 测试前端操作响应时间
   - 测试图表渲染时间

### 测试配置

**后端测试配置**:
```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
asyncio_mode = "auto"
```

**前端测试配置**:
```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    environment: 'jsdom',
    setupFiles: ['./src/views/business-analysis-v3/__tests__/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html']
    }
  }
})
```


## 性能优化

### 后端性能优化

#### 1. 数据库查询优化

**索引策略**:
```sql
-- 已有索引
CREATE INDEX idx_orders_customer_created ON orders(customer_global_id, created_at);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_customers_source ON customers(customer_source);
CREATE INDEX idx_customers_group ON customers(group_type);
```

**查询优化**:
- 只查询需要的列
- 使用 INNER JOIN 而不是 LEFT JOIN
- 在数据库层面做类型转换
- 使用 LIMIT 限制返回行数（如果需要）

#### 2. Polars 性能优化

**使用 ConnectorX**:
```python
import connectorx as cx

# 直接从数据库读取到 Polars（零拷贝）
df = pl.from_pandas(cx.read_sql(
    query_sql,
    settings.database.connectorx_url,
    return_type="pandas"
))
```

**性能提升**:
- 数据加载速度提升 5-10 倍
- 减少内存复制
- 并行读取

**链式 API**:
```python
# 使用 Polars 链式 API
df = (
    df
    .with_columns([
        pl.col("order_created_date").dt.year().alias("year"),
        pl.col("order_created_date").dt.month().alias("month")
    ])
    .group_by(["customer_source", "year"])
    .agg([
        pl.col("order_id").n_unique().alias("order_count"),
        pl.col("order_amount").sum().alias("total_amount")
    ])
)
```

#### 3. HTTP 缓存优化

**ETag 生成**:
```python
import hashlib

def generate_etag(data: bytes) -> str:
    """生成 ETag（基于数据内容的 MD5）"""
    return hashlib.md5(data).hexdigest()
```

**缓存头设置**:
```python
response.headers["ETag"] = etag
response.headers["Cache-Control"] = "max-age=3600, must-revalidate"
```

**条件请求处理**:
```python
if_none_match = request.headers.get("If-None-Match")
if if_none_match == etag:
    return Response(status_code=304)
```

### 前端性能优化

#### 1. Perspective 懒加载

**动态导入**:
```typescript
// 只在需要时加载 Perspective
const loadPerspective = async () => {
  const perspective = await import('@finos/perspective')
  return perspective.default.worker()
}
```

**单例模式**:
```typescript
let perspectiveWorker: Worker | null = null

const getWorker = async () => {
  if (!perspectiveWorker) {
    const perspective = await import('@finos/perspective')
    perspectiveWorker = perspective.default.worker()
  }
  return perspectiveWorker
}
```

#### 2. 组件懒加载

**异步组件**:
```typescript
const DataExplorer = defineAsyncComponent(() =>
  import('./components/DataExplorer.vue')
)

const BusinessAnalysisModel = defineAsyncComponent(() =>
  import('./components/BusinessAnalysisModel.vue')
)
```

#### 3. 数据缓存

**前端缓存策略**:
```typescript
const cache = new Map<string, any>()

const fetchWithCache = async (url: string) => {
  if (cache.has(url)) {
    return cache.get(url)
  }
  
  const response = await fetch(url)
  const data = await response.json()
  cache.set(url, data)
  
  return data
}
```

#### 4. 虚拟滚动

**Perspective 内置虚拟滚动**:
- 只渲染可见区域
- 支持 1000W 行数据
- 流畅滚动

**Naive UI DataTable 虚拟滚动**:
```vue
<n-data-table
  :columns="columns"
  :data="data"
  :virtual-scroll="true"
  :max-height="600"
/>
```

### 性能监控

#### 1. 后端性能监控

**日志记录**:
```python
import time
from loguru import logger

start_time = time.time()
# ... 执行操作
execution_time = time.time() - start_time
logger.info(f"操作完成，耗时 {execution_time:.2f} 秒")
```

**性能指标**:
- 数据库查询时间
- Polars 计算时间
- Arrow 序列化时间
- 总响应时间

#### 2. 前端性能监控

**Performance API**:
```typescript
const startTime = performance.now()
// ... 执行操作
const endTime = performance.now()
console.log(`操作耗时: ${endTime - startTime}ms`)
```

**性能指标**:
- 组件加载时间
- 数据加载时间
- 图表渲染时间
- 操作响应时间

### 性能目标

| 指标 | 目标 | 当前 |
|------|------|------|
| Arrow 数据加载 | < 2 秒（50W 行） | ~1.5 秒 |
| JSON 数据加载 | < 2 秒（50W 行） | ~1.0 秒 |
| 前端操作响应 | < 100ms | ~50ms |
| 图表渲染 | < 500ms | ~300ms |
| 缓存命中响应 | < 50ms | ~20ms |


## 部署架构

### 开发环境

```
┌─────────────────────────────────────────────────────────────┐
│                      开发环境                                 │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  前端开发服务器 (Vite)                                         │
│  http://localhost:5173                                       │
│         │                                                     │
│         │ Proxy                                               │
│         ↓                                                     │
│  后端开发服务器 (Uvicorn)                                      │
│  http://localhost:8000                                       │
│         │                                                     │
│         ↓                                                     │
│  PostgreSQL 数据库                                            │
│  localhost:5432                                              │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**启动命令**:
```bash
# 后端
cd OneManage
pdm run uvicorn main:app --reload

# 前端
cd OneManage_web
pnpm dev
```

### 生产环境

```
┌─────────────────────────────────────────────────────────────┐
│                      生产环境                                 │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Nginx (反向代理)                                             │
│  https://yourdomain.com                                      │
│         │                                                     │
│         ├─ /api/* ──────────→ FastAPI (Gunicorn + Uvicorn)  │
│         │                     http://localhost:8000          │
│         │                            │                        │
│         └─ /* ──────────────→ 静态文件 (Vue 3 Build)          │
│                                       │                        │
│                                       ↓                        │
│                              PostgreSQL (RDS)                 │
│                              阿里云 RDS                        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**部署步骤**:

1. **构建前端**:
```bash
cd OneManage_web
pnpm build
```

2. **部署后端**:
```bash
cd OneManage
pdm install --prod
pdm run gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker
```

3. **配置 Nginx**:
```nginx
server {
    listen 80;
    server_name yourdomain.com;
    
    # 前端静态文件
    location / {
        root /var/www/onemanage-web/dist;
        try_files $uri $uri/ /index.html;
    }
    
    # 后端 API
    location /api/ {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## 安全考虑

### 数据安全

1. **SQL 注入防护**
   - 使用 SQLAlchemy 参数化查询
   - 不拼接 SQL 字符串

2. **XSS 防护**
   - Vue 3 自动转义
   - 不使用 v-html

3. **CSRF 防护**
   - 使用 CORS 限制来源
   - 使用 CSRF Token（如果需要）

### API 安全

1. **认证和授权**
   - 使用 JWT Token（如果需要）
   - 实现权限控制（后续版本）

2. **速率限制**
   - 使用 slowapi 限制请求频率
   - 防止 DDoS 攻击

3. **数据验证**
   - 使用 Pydantic 验证所有输入
   - 拒绝无效请求

### 缓存安全

1. **缓存污染防护**
   - 使用 ETag 验证缓存有效性
   - 设置合理的缓存时间

2. **敏感数据保护**
   - 不缓存敏感数据
   - 使用 HTTPS 传输

## 监控和日志

### 日志策略

**后端日志**:
```python
from loguru import logger

# 配置日志
logger.add(
    "logs/business_analysis_{time}.log",
    rotation="1 day",
    retention="30 days",
    level="INFO"
)

# 记录关键操作
logger.info(f"用户 {user_id} 执行分析，参数: {params}")
logger.info(f"数据加载完成，耗时 {execution_time:.2f} 秒")
logger.error(f"数据库查询失败: {error}")
```

**前端日志**:
```typescript
// 开发环境：console.log
// 生产环境：发送到日志服务

const logError = (error: Error, context: string) => {
  if (import.meta.env.DEV) {
    console.error(`[${context}]`, error)
  } else {
    // 发送到日志服务（如 Sentry）
    sendToLogService({ error, context })
  }
}
```

### 监控指标

**关键指标**:
- API 响应时间
- 数据库查询时间
- 缓存命中率
- 错误率
- 并发用户数

**告警规则**:
- API 响应时间 > 5 秒
- 错误率 > 5%
- 数据库连接失败
- 内存使用 > 80%

## 维护和扩展

### 代码维护

1. **代码规范**
   - 遵循项目规范
   - 使用 ESLint 和 Black
   - 代码审查

2. **文档维护**
   - 更新 API 文档
   - 更新用户手册
   - 记录变更日志

3. **测试维护**
   - 保持测试覆盖率 > 80%
   - 定期运行测试
   - 修复失败的测试

### 功能扩展

**短期扩展**:
- 添加更多预定义分析模型
- 优化缓存策略
- 添加数据导出功能

**中期扩展**:
- 实现视图分享功能
- 添加权限控制
- 实现实时数据流

**长期扩展**:
- 引入物化视图
- 添加 AI 驱动的洞察
- 实现协作功能

### 性能扩展

**数据量增长应对**:

| 数据量 | 策略 |
|--------|------|
| < 50W 行 | 当前架构 |
| 50W - 100W 行 | 引入 Redis 缓存 |
| 100W - 500W 行 | 使用物化视图 |
| > 500W 行 | 分库分表 + 数据归档 |

## UI/UX 重构设计

### 重构目标

将当前的垂直布局改为专业的数据分析工具风格，采用左右分栏布局，提升用户体验和操作效率。

### 新布局架构

```
┌─────────────────────────────────────────────────────────────┐
│  顶部导航栏                                                    │
│  [业务分析平台] [数据加载区域] [数据探索/业务分析 Tab]          │
├──────────┬──────────────────────────────────────────────────┤
│          │                                                    │
│  左侧    │                                                    │
│  配置    │              主内容区域                             │
│  面板    │                                                    │
│          │          (图表 + 表格 + 结果展示)                   │
│  - 模型  │                                                    │
│  - 参数  │                                                    │
│  - 配置  │                                                    │
│          │                                                    │
│  [分析]  │                                                    │
│          │                                                    │
└──────────┴──────────────────────────────────────────────────┘
```

### 关键改进点

#### 1. 顶部数据加载区域
**当前**：配置面板内的日期选择
**改进**：
- 独立的顶部工具栏
- 日期范围选择器（开始日期 → 结束日期）
- "加载数据"按钮（绿色主题色）
- Tab 切换（数据探索 / 业务分析）

#### 2. 左侧配置面板
**当前**：居中的卡片式配置
**改进**：
- 固定宽度侧边栏（280-320px）
- 分组展示配置项
- 可折叠的配置区域
- 底部固定"开始分析"按钮

**配置面板结构**：
```
┌─────────────────┐
│ RFM客户价值分析  │ ← 当前选中模型
├─────────────────┤
│ 最近购买 (R)     │
│ ○ 订单日期       │
│ ○ 支付日期       │
├─────────────────┤
│ 购买频率 (F)     │
│ ○ 订单数量       │
│ ○ 订单金额       │
├─────────────────┤
│ 购买金额 (M)     │
│ ○ 总金额         │
│ ○ 平均金额       │
├─────────────────┤
│ 分组方式         │
│ ○ 3分组(简化版)  │
│ ● 5分组(标准版)  │
│ ○ 自定义分组     │
├─────────────────┤
│                 │
│  [▶ 开始分析]   │ ← 底部固定按钮
└─────────────────┘
```

#### 3. 主内容区域
**当前**：下方显示结果
**改进**：
- 右侧占据主要空间
- 更大的可视化区域
- 结果和图表同屏显示
- 支持全屏模式

#### 4. 交互优化
- 配置和结果同屏显示，无需滚动
- 左侧配置变更实时预览
- 支持键盘快捷键（Enter 开始分析）
- 响应式布局适配不同屏幕

### 组件重构计划

#### 需要重构的组件

1. **index.vue（主页面）**
   - 添加顶部工具栏
   - 实现左右分栏布局
   - 移除 Tab 切换器（移到顶部）

2. **BusinessAnalysisModel.vue**
   - 改为左右分栏布局
   - 左侧：配置面板
   - 右侧：结果展示

3. **ConfigPanel.vue**
   - 改为侧边栏样式
   - 添加分组折叠功能
   - 底部固定按钮

4. **新增：TopToolbar.vue**
   - 数据加载区域
   - Tab 切换
   - 全局操作按钮

#### 保持不变的组件

- ChartsPanel.vue（图表面板）
- DataTable.vue（数据表格）
- ModelSelector.vue（模型选择器）
- 所有分析模型组件（RFM、复购、帕累托、波士顿矩阵）
- 所有 composables（业务逻辑不变）

### 样式设计

#### 颜色方案（保持白色主题）
- 主背景：#f5f7fa（浅灰）
- 卡片背景：#ffffff（白色）
- 主题色：#18a058（绿色，用于按钮和强调）
- 边框：#e0e0e6（浅灰边框）
- 文字：#333333（深灰）

#### 布局尺寸
- 左侧面板宽度：300px
- 顶部工具栏高度：64px
- 内容区域间距：24px
- 卡片圆角：8px

### 响应式设计

#### 桌面端（>1200px）
- 左右分栏布局
- 左侧固定 300px
- 右侧自适应

#### 平板端（768px-1200px）
- 左侧面板可折叠
- 点击展开/收起
- 右侧占据主要空间

#### 移动端（<768px）
- 垂直布局
- 配置面板在顶部
- 结果在下方
- 全屏显示

### 性能优化

- 使用 CSS Grid 布局（性能更好）
- 虚拟滚动（大数据表格）
- 懒加载图表组件
- 防抖处理配置变更

### 可访问性

- 支持键盘导航
- ARIA 标签
- 高对比度模式
- 屏幕阅读器支持

## 总结

本设计文档描述了基于混合架构的业务分析功能，核心思路是：

1. **清晰分离两个场景**
   - 场景 A：数据探索（Perspective）
   - 场景 B：业务分析模型（ECharts）

2. **各自使用最适合的技术栈**
   - 数据探索：Arrow IPC + Perspective（自由拖拽）
   - 业务分析：JSON + ECharts（固定图表）

3. **避免不必要的复杂度**
   - 不在 Perspective 和 ECharts 之间做数据转换
   - 不让 Perspective 成为"中转站"
   - 保持代码简洁清晰

4. **提供优秀的用户体验**
   - 数据探索：灵活、交互式、适合探索性分析
   - 业务分析：简单、直接、适合日常监控
   - **UI 重构**：专业的左右分栏布局，配置和结果同屏显示

这个架构设计充分考虑了性能、可维护性和可扩展性，能够满足当前和未来的业务需求。
