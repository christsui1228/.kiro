# 业务分析功能设计文档

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                    前端层 (Vue 3)                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  BusinessAnalysis.vue (主页面)                   │  │
│  │  ├─ ConfigPanel (配置面板 - Naive UI)            │  │
│  │  ├─ DataGrid (数据表格 - Revo Grid)              │  │
│  │  └─ Charts (图表展示 - ECharts)                  │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                          ↓ HTTP API
┌─────────────────────────────────────────────────────────┐
│                  后端层 (FastAPI)                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  routes.py (API 路由)                            │  │
│  │  ├─ POST /api/analytics/customer-orders          │  │
│  │  └─ GET  /api/analytics/metadata                 │  │
│  └──────────────────────────────────────────────────┘  │
│                          ↓                              │
│  ┌──────────────────────────────────────────────────┐  │
│  │  analysis.py (Polars 分析引擎)                   │  │
│  │  ├─ CustomerOrderAnalyzer                        │  │
│  │  ├─ _load_data() - 数据加载                      │  │
│  │  ├─ _add_time_column() - 时间处理                │  │
│  │  └─ _aggregate() - 聚合计算                      │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                          ↓ SQL Query
┌─────────────────────────────────────────────────────────┐
│              数据库层 (PostgreSQL)                       │
│  ┌──────────────┐         ┌──────────────┐            │
│  │  customers   │ 1:N     │   orders     │            │
│  │  ─────────── │────────→│  ──────────  │            │
│  │  customer_id │         │  customer_id │            │
│  │  source      │         │  order_id    │            │
│  │  group_type  │         │  amount      │            │
│  └──────────────┘         │  created_at  │            │
│                           └──────────────┘            │
└─────────────────────────────────────────────────────────┘
```

### 模块结构

```
features/business_analysis/
├── __init__.py
├── models.py           # 数据模型（可选，用于保存分析配置）
├── schemas.py          # Pydantic 请求/响应模型
├── analysis.py         # Polars 分析引擎核心逻辑
├── routes.py           # FastAPI 路由定义
└── constants.py        # 常量定义（维度、指标映射）

前端（OneManage_web）:
OneManage_web/src/views/business-analysis/
├── index.vue                    # 主页面入口
├── components/
│   ├── ConfigPanel.vue          # 配置面板
│   ├── DataGrid.vue             # Revo Grid 表格
│   └── Charts.vue               # ECharts 图表
└── composables/
    └── useCustomerOrderAnalysis.ts  # 分析逻辑和状态管理
```

## 数据模型设计

### 数据库表结构

**customers 表**（已存在）：
```sql
CREATE TABLE customers (
    customer_global_id UUID PRIMARY KEY,
    customer_source VARCHAR(10),      -- '流量' | '推荐'
    group_type VARCHAR(10),           -- '家庭' | '公司' | '其他'
    customer_nickname VARCHAR(64),
    shop_id UUID,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

**orders 表**（已存在）：
```sql
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_global_id UUID REFERENCES customers(customer_global_id),
    order_amount DECIMAL(10, 2),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

### 数据库索引设计

```sql
-- 订单表索引（性能优化）
CREATE INDEX idx_orders_customer_created 
ON orders(customer_global_id, created_at);

CREATE INDEX idx_orders_created_at 
ON orders(created_at DESC);

-- 客户表索引
CREATE INDEX idx_customers_source 
ON customers(customer_source);

CREATE INDEX idx_customers_group 
ON customers(group_type);
```

### API 数据模型

**请求模型**：
```python
from pydantic import BaseModel, Field
from datetime import date
from typing import Literal, Optional

class CustomerOrderAnalysisRequest(BaseModel):
    """客户订单分析请求"""
    
    group_by: Literal["customer_source", "group_type"] = Field(
        default="customer_source",
        description="分组维度"
    )
    
    time_grain: Optional[Literal["year", "month"]] = Field(
        default="month",
        description="时间粒度，null 表示不分组"
    )
    
    date_range: dict = Field(
        description="时间范围",
        example={"start": "2024-01-01", "end": "2024-12-31"}
    )
    
    class Config:
        json_schema_extra = {
            "example": {
                "group_by": "customer_source",
                "time_grain": "month",
                "date_range": {
                    "start": "2024-01-01",
                    "end": "2024-12-31"
                }
            }
        }
```

**响应模型**：
```python
from typing import List, Optional

class CustomerOrderAnalysisData(BaseModel):
    """单条分析数据"""
    
    dimension_value: str = Field(description="维度值，如'流量'")
    time_period: Optional[str] = Field(default=None, description="时间周期，如'2024-01'")
    order_count: int = Field(description="订单数量")
    total_amount: float = Field(description="累计金额")
    avg_amount: float = Field(description="平均订单金额")

class CustomerOrderAnalysisResponse(BaseModel):
    """分析结果响应"""
    
    data: List[CustomerOrderAnalysisData]
    summary: dict = Field(description="汇总信息")
    execution_time: float = Field(description="执行时间（秒）")
    
    class Config:
        json_schema_extra = {
            "example": {
                "data": [
                    {
                        "dimension_value": "流量",
                        "time_period": "2024-01",
                        "order_count": 320,
                        "total_amount": 150000.00,
                        "avg_amount": 468.75
                    }
                ],
                "summary": {
                    "total_orders": 5200,
                    "total_amount": 2390000.00,
                    "avg_amount": 459.62
                },
                "execution_time": 1.23
            }
        }
```

## 核心组件设计

### 1. Polars 分析引擎

**CustomerOrderAnalyzer 类**：
```python
import polars as pl
from datetime import datetime
from typing import Optional, List

class CustomerOrderAnalyzer:
    """客户订单分析引擎"""
    
    def __init__(self, db_connection_uri: str):
        self.db_uri = db_connection_uri
    
    async def analyze(
        self,
        group_by: str,
        time_grain: Optional[str],
        start_date: date,
        end_date: date
    ) -> pl.DataFrame:
        """
        执行分析
        
        流程：
        1. 从数据库加载数据（只查询需要的字段）
        2. 添加时间列（如果需要）
        3. 按维度聚合
        4. 计算指标
        5. 排序返回
        """
        
        # 1. 加载数据
        df = await self._load_data(group_by, start_date, end_date)
        
        # 2. 添加时间列
        if time_grain:
            df = self._add_time_column(df, time_grain)
        
        # 3. 聚合计算
        result = self._aggregate(df, group_by, time_grain)
        
        return result
    
    async def _load_data(
        self, 
        group_by: str, 
        start_date: date, 
        end_date: date
    ) -> pl.DataFrame:
        """
        从数据库加载数据
        
        优化策略：
        - 只查询需要的字段
        - 在数据库层面先过滤时间范围
        - 使用索引加速查询
        """
        
        query = f"""
        SELECT 
            c.{group_by} as dimension_value,
            o.order_id,
            o.order_amount,
            o.created_at
        FROM customers c
        INNER JOIN orders o ON c.customer_global_id = o.customer_global_id
        WHERE o.created_at BETWEEN :start_date AND :end_date
        """
        
        # Polars 直接读取数据库
        df = pl.read_database(
            query, 
            self.db_uri,
            params={
                "start_date": start_date,
                "end_date": end_date
            }
        )
        
        return df
    
    def _add_time_column(
        self, 
        df: pl.DataFrame, 
        time_grain: str
    ) -> pl.DataFrame:
        """
        添加时间列
        
        支持的粒度：
        - year: 2024
        - month: 2024-01
        """
        
        if time_grain == "year":
            return df.with_columns(
                pl.col("created_at").dt.year().cast(pl.Utf8).alias("time_period")
            )
        elif time_grain == "month":
            return df.with_columns(
                pl.col("created_at").dt.strftime("%Y-%m").alias("time_period")
            )
        
        return df
    
    def _aggregate(
        self,
        df: pl.DataFrame,
        group_by: str,
        time_grain: Optional[str]
    ) -> pl.DataFrame:
        """
        聚合计算指标
        
        指标：
        - order_count: 订单数量（去重）
        - total_amount: 累计金额
        - avg_amount: 平均订单金额
        """
        
        # 确定分组列
        group_cols = ["dimension_value"]
        if time_grain:
            group_cols.append("time_period")
        
        # 聚合计算
        result = df.group_by(group_cols).agg([
            pl.col("order_id").n_unique().alias("order_count"),
            pl.col("order_amount").sum().alias("total_amount"),
            pl.col("order_amount").mean().alias("avg_amount")
        ])
        
        # 排序
        result = result.sort(group_cols)
        
        # 格式化金额（保留 2 位小数）
        result = result.with_columns([
            pl.col("total_amount").round(2),
            pl.col("avg_amount").round(2)
        ])
        
        return result
```

### 2. API 路由设计

**routes.py**：
```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from datetime import datetime
import time

router = APIRouter(prefix="/api/analytics", tags=["业务分析"])

@router.get("/metadata")
async def get_analysis_metadata():
    """
    获取分析元数据
    
    返回可用的维度、指标、时间粒度选项
    """
    return {
        "dimensions": [
            {"value": "customer_source", "label": "客户来源"},
            {"value": "group_type", "label": "客户分组"}
        ],
        "time_grains": [
            {"value": None, "label": "不分组"},
            {"value": "year", "label": "按年"},
            {"value": "month", "label": "按月"}
        ]
    }

@router.post("/customer-orders", response_model=CustomerOrderAnalysisResponse)
async def analyze_customer_orders(
    request: CustomerOrderAnalysisRequest,
    db: AsyncSession = Depends(get_db)
):
    """
    客户订单分析
    
    执行流程：
    1. 验证请求参数
    2. 调用 Polars 分析引擎
    3. 计算汇总信息
    4. 返回结果
    """
    
    start_time = time.time()
    
    try:
        # 1. 参数验证
        start_date = datetime.strptime(request.date_range["start"], "%Y-%m-%d").date()
        end_date = datetime.strptime(request.date_range["end"], "%Y-%m-%d").date()
        
        if end_date < start_date:
            raise HTTPException(status_code=400, detail="结束日期不能早于开始日期")
        
        # 2. 执行分析
        analyzer = CustomerOrderAnalyzer(db_connection_uri=settings.database_url)
        result_df = await analyzer.analyze(
            group_by=request.group_by,
            time_grain=request.time_grain,
            start_date=start_date,
            end_date=end_date
        )
        
        # 3. 转换为响应格式
        data = result_df.to_dicts()
        
        # 4. 计算汇总
        summary = {
            "total_orders": result_df["order_count"].sum(),
            "total_amount": float(result_df["total_amount"].sum()),
            "avg_amount": float(result_df["avg_amount"].mean())
        }
        
        execution_time = time.time() - start_time
        
        return CustomerOrderAnalysisResponse(
            data=data,
            summary=summary,
            execution_time=round(execution_time, 2)
        )
        
    except Exception as e:
        logger.error(f"分析失败: {str(e)}")
        raise HTTPException(status_code=500, detail=f"分析失败: {str(e)}")
```

### 3. 前端组件设计

**ConfigPanel.vue（配置面板）**：
```vue
<template>
  <n-card title="分析配置">
    <n-space vertical>
      <!-- 分组维度 -->
      <n-form-item label="分组维度">
        <n-radio-group v-model:value="config.groupBy">
          <n-radio value="customer_source">客户来源</n-radio>
          <n-radio value="group_type">客户分组</n-radio>
        </n-radio-group>
      </n-form-item>
      
      <!-- 时间粒度 -->
      <n-form-item label="时间粒度">
        <n-radio-group v-model:value="config.timeGrain">
          <n-radio :value="null">不分组</n-radio>
          <n-radio value="year">按年</n-radio>
          <n-radio value="month">按月</n-radio>
        </n-radio-group>
      </n-form-item>
      
      <!-- 时间范围 -->
      <n-form-item label="时间范围">
        <n-date-picker
          v-model:value="config.dateRange"
          type="daterange"
          clearable
          :default-value="defaultDateRange"
        />
      </n-form-item>
      
      <!-- 操作按钮 -->
      <n-space>
        <n-button type="primary" @click="handleAnalyze" :loading="loading">
          开始分析
        </n-button>
        <n-button @click="handleReset">重置</n-button>
      </n-space>
    </n-space>
  </n-card>
</template>
```

**DataGrid.vue（数据表格）**：
```vue
<template>
  <n-card title="分析结果">
    <revo-grid
      :source="tableData"
      :columns="tableColumns"
      theme="compact"
      :resize="true"
      :filter="true"
      :range="true"
      style="height: 500px;"
    />
  </n-card>
</template>

<script setup lang="ts">
import { RevoGrid } from '@revolist/vue3-datagrid'
import { computed } from 'vue'

const props = defineProps<{
  data: any[]
  groupBy: string
  timeGrain: string | null
}>()

const tableColumns = computed(() => {
  const cols = [
    {
      prop: 'dimension_value',
      name: props.groupBy === 'customer_source' ? '客户来源' : '客户分组',
      size: 150,
      pin: 'colPinStart'
    }
  ]
  
  if (props.timeGrain) {
    cols.push({
      prop: 'time_period',
      name: '时间周期',
      size: 120
    })
  }
  
  cols.push(
    {
      prop: 'order_count',
      name: '订单数量',
      size: 120,
      columnType: 'numeric'
    },
    {
      prop: 'total_amount',
      name: '累计金额',
      size: 150,
      columnType: 'numeric'
    },
    {
      prop: 'avg_amount',
      name: '平均金额',
      size: 150,
      columnType: 'numeric'
    }
  )
  
  return cols
})

const tableData = computed(() => props.data)
</script>
```

**Charts.vue（图表展示）**：
```vue
<template>
  <n-card title="图表分析">
    <n-tabs v-model:value="activeChart">
      <n-tab-pane name="bar" tab="柱状图">
        <div ref="barChartRef" style="width: 100%; height: 400px;"></div>
      </n-tab-pane>
      
      <n-tab-pane name="line" tab="折线图" v-if="timeGrain">
        <div ref="lineChartRef" style="width: 100%; height: 400px;"></div>
      </n-tab-pane>
    </n-tabs>
  </n-card>
</template>

<script setup lang="ts">
import * as echarts from 'echarts'
import { ref, watch, onMounted } from 'vue'

const props = defineProps<{
  data: any[]
  timeGrain: string | null
}>()

const barChartRef = ref()
const lineChartRef = ref()

const renderBarChart = () => {
  const chart = echarts.init(barChartRef.value)
  
  const option = {
    tooltip: { trigger: 'axis' },
    legend: { data: ['订单数量', '累计金额'] },
    xAxis: {
      type: 'category',
      data: props.data.map(d => d.dimension_value)
    },
    yAxis: [
      { type: 'value', name: '订单数量' },
      { type: 'value', name: '累计金额' }
    ],
    series: [
      {
        name: '订单数量',
        type: 'bar',
        data: props.data.map(d => d.order_count)
      },
      {
        name: '累计金额',
        type: 'bar',
        yAxisIndex: 1,
        data: props.data.map(d => d.total_amount)
      }
    ]
  }
  
  chart.setOption(option)
}

const renderLineChart = () => {
  if (!props.timeGrain) return
  
  const chart = echarts.init(lineChartRef.value)
  
  const option = {
    tooltip: { trigger: 'axis' },
    legend: { data: ['订单数量', '累计金额'] },
    xAxis: {
      type: 'category',
      data: props.data.map(d => d.time_period)
    },
    yAxis: [
      { type: 'value', name: '订单数量' },
      { type: 'value', name: '累计金额' }
    ],
    series: [
      {
        name: '订单数量',
        type: 'line',
        data: props.data.map(d => d.order_count)
      },
      {
        name: '累计金额',
        type: 'line',
        yAxisIndex: 1,
        data: props.data.map(d => d.total_amount)
      }
    ]
  }
  
  chart.setOption(option)
}

watch(() => props.data, () => {
  renderBarChart()
  if (props.timeGrain) {
    renderLineChart()
  }
})
</script>
```

## 性能优化策略

### 1. 数据库层优化

**索引策略**：
- 在 `orders(customer_global_id, created_at)` 上创建复合索引
- 在 `orders(created_at)` 上创建单列索引
- 在 `customers(customer_source)` 和 `customers(group_type)` 上创建索引

**查询优化**：
- 只查询需要的字段，避免 `SELECT *`
- 在数据库层面先过滤时间范围
- 使用 INNER JOIN 而非 LEFT JOIN（确保有订单的客户）

### 2. Polars 计算优化

**内存优化**：
- 使用 Polars 的列式存储，减少内存占用
- 及时释放不需要的中间结果

**并行计算**：
- Polars 自动并行处理，无需手动配置
- 利用多核 CPU 加速聚合计算

### 3. 前端渲染优化

**虚拟滚动**：
- Revo Grid 自带虚拟滚动，支持大数据量

**按需渲染**：
- 图表只在切换到对应 tab 时才渲染
- 使用 `v-if` 而非 `v-show` 控制图表显示

**防抖处理**：
- 用户修改配置后，延迟 500ms 再发起请求

## 错误处理设计

### 后端错误处理

```python
# 统一错误响应格式
{
    "detail": "错误描述",
    "error_code": "ERROR_CODE",
    "timestamp": "2024-11-24T10:00:00Z"
}

# 错误类型
- 400: 参数错误（日期范围无效、维度不支持等）
- 404: 数据不存在（所选时间范围内无数据）
- 500: 服务器错误（数据库连接失败、计算异常等）
- 504: 超时错误（查询超过 10 秒）
```

### 前端错误处理

```typescript
// 错误提示策略
try {
  const result = await analyzeCustomerOrders(params)
  // 处理结果
} catch (error) {
  if (error.response?.status === 400) {
    message.error('参数错误，请检查配置')
  } else if (error.response?.status === 404) {
    message.warning('所选时间范围内无数据')
  } else if (error.response?.status === 500) {
    message.error('系统繁忙，请稍后重试')
  } else {
    message.error('网络连接失败，请检查网络')
  }
}
```

## 测试策略

### 后端测试（不在 Spec 中实现）

- 单元测试：测试 Polars 分析引擎的各个方法
- 集成测试：测试 API 接口的完整流程
- 性能测试：验证 50W 订单的查询性能

### 前端测试（不在 Spec 中实现）

- 组件测试：测试配置面板、表格、图表的渲染
- 集成测试：测试完整的分析流程
- E2E 测试：模拟用户操作流程

## 部署考虑

### 数据库准备

```sql
-- 创建索引（部署前执行）
CREATE INDEX CONCURRENTLY idx_orders_customer_created 
ON orders(customer_global_id, created_at);

CREATE INDEX CONCURRENTLY idx_orders_created_at 
ON orders(created_at DESC);

CREATE INDEX CONCURRENTLY idx_customers_source 
ON customers(customer_source);

CREATE INDEX CONCURRENTLY idx_customers_group 
ON customers(group_type);
```

### 环境配置

```bash
# .env.development
ENV=development
DB_HOST=localhost
DB_PORT=5432
DB_NAME=onemanage_dev

# .env.production
ENV=production
DB_HOST=rds-xxx.aliyuncs.com
DB_PORT=5432
DB_NAME=onemanage_prod
```

## 扩展性设计

### 未来扩展方向

1. **新增维度**：
   - 店铺维度（shop_id）
   - 地区维度（需要增加地区字段）

2. **新增指标**：
   - 成本相关（毛利、快递成本）
   - 客户价值（复购率、生命周期价值）

3. **新增功能**：
   - 导出 Excel/CSV
   - 报表保存和分享
   - 定时报表推送

### 扩展接口设计

```python
# 预留扩展接口
class BaseAnalyzer(ABC):
    """分析器基类"""
    
    @abstractmethod
    async def analyze(self, **kwargs) -> pl.DataFrame:
        """执行分析"""
        pass

class CustomerOrderAnalyzer(BaseAnalyzer):
    """客户订单分析器"""
    pass

class CustomerValueAnalyzer(BaseAnalyzer):
    """客户价值分析器（未来扩展）"""
    pass
```

## 技术债务

### 已知限制

1. **时间范围限制**：当前限制 3 年，未来可能需要支持更长时间
2. **权限控制**：当前无权限控制，未来需要按店铺隔离数据
3. **缓存策略**：当前无缓存，未来可能需要内存缓存优化

### 优化计划

1. **第一阶段**：完成 MVP，验证功能和性能
2. **第二阶段**：根据实际使用情况优化性能
3. **第三阶段**：增加成本分析和高级指标
