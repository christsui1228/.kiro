# 业务分析功能升级设计文档（Perspective 4.0 架构）

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    前端层 (Vue 3)                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  浏览器缓存 (HTTP Cache)                             │  │
│  │  ├─ Cache-Control: max-age=3600                     │  │
│  │  ├─ ETag 支持                                        │  │
│  │  └─ 条件请求 (If-None-Match)                        │  │
│  └──────────────────────────────────────────────────────┘  │
│                          ↓                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Perspective 4.0 Viewer (WebAssembly)                │  │
│  │  ├─ Datagrid（数据表格）                              │  │
│  │  ├─ Pivot Table（透视表）                            │  │
│  │  ├─ Y Line/Bar（内置图表）                           │  │
│  │  └─ 动态分组、聚合、筛选                              │  │
│  └──────────────────────────────────────────────────────┘  │
│                          ↕                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  ECharts 6.0（高级图表）                              │  │
│  │  ├─ 从 Perspective 获取数据                           │  │
│  │  ├─ 自定义图表配置                                    │  │
│  │  └─ 实时数据同步                                      │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                          ↓ Arrow IPC (零拷贝)
┌─────────────────────────────────────────────────────────────┐
│                   应用层 (FastAPI)                           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Polars 分析引擎                                      │  │
│  │  ├─ 直接查询原始表                                    │  │
│  │  ├─ 返回明细数据（不聚合）                            │  │
│  │  └─ 导出 Arrow IPC                                   │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                          ↓ SQL Query
┌─────────────────────────────────────────────────────────────┐
│                 数据库层 (PostgreSQL)                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  原始表 + 索引                                        │  │
│  │  ├─ customers (10W 行)                               │  │
│  │  ├─ orders (30W 行)                                  │  │
│  │  ├─ customer_groups                                  │  │
│  │  └─ 索引优化（已有）                                  │  │
│  │                                                       │  │
│  │  注：当前数据量 30W 行，直接查询原始表即可满足性能要求  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 数据流转

```
1. 用户打开分析页面
   ↓
2. 前端请求 Arrow 数据 (POST /api/analytics/customer-orders/arrow)
   ├─ 带 If-None-Match 头（如果有 ETag）
   ↓
3. 后端检查 ETag
   ├─ 匹配 → 返回 304 Not Modified（浏览器使用缓存）
   └─ 不匹配或无 ETag ↓
4. Polars 查询数据库原始表 (30W 行, 1-2s)
   ↓
5. 转换为 Arrow IPC 格式
   ↓
6. 生成 ETag（基于数据内容）
   ↓
7. 返回给前端（带 Cache-Control 和 ETag 头）
   ↓
8. 浏览器缓存 Arrow 数据
   ↓
9. Perspective 加载 Arrow 数据 (200-500ms)
   ↓
10. 用户自由操作 (分组、聚合、筛选) - 毫秒级响应
   ↓
11. (可选) ECharts 从 Perspective 获取数据并渲染
```

## 核心组件设计

### 1. 后端：Arrow IPC 导出

#### 路由设计

```python
# features/business_analysis/routes.py

@router.post("/customer-orders/arrow")
async def analyze_customer_orders_arrow(
    request: CustomerOrderAnalysisRequest,
    db: AsyncSession = Depends(get_db),
    if_none_match: str | None = Header(None)
) -> Response:
    """
    返回 Arrow IPC 格式的分析数据
    
    优化策略：
    1. 直接查询原始表
    2. 转换为 Arrow IPC
    3. 使用 HTTP 缓存（ETag + Cache-Control）
    """
    
    # 1. 查询数据
    df = await customer_order_service.get_raw_data(db, request)
    
    # 2. 转换为 Arrow IPC
    arrow_table = df.to_arrow()
    sink = pa.BufferOutputStream()
    writer = pa.ipc.new_stream(sink, arrow_table.schema)
    writer.write_table(arrow_table)
    writer.close()
    
    arrow_bytes = sink.getvalue().to_pybytes()
    
    # 3. 生成 ETag（基于数据内容的哈希）
    import hashlib
    etag = hashlib.md5(arrow_bytes).hexdigest()
    
    # 4. 检查 ETag 是否匹配
    if if_none_match and if_none_match == f'"{etag}"':
        logger.info("Cache HIT: Browser (304)")
        return Response(
            status_code=304,
            headers={"ETag": f'"{etag}"'}
        )
    
    # 5. 返回数据（带缓存头）
    return Response(
        content=arrow_bytes,
        media_type="application/vnd.apache.arrow.stream",
        headers={
            "ETag": f'"{etag}"',
            "Cache-Control": "private, max-age=3600",  # 缓存 1 小时
            "X-Content-Hash": etag
        }
    )
```

#### Service 层设计

```python
# features/business_analysis/services/customer_order_service.py

class CustomerOrderService:
    """客户订单分析服务"""
    
    async def get_raw_data(
        self,
        db: AsyncSession,
        request: CustomerOrderAnalysisRequest
    ) -> pl.DataFrame:
        """
        获取原始数据（不做聚合）
        
        返回所有订单的明细数据，让前端 Perspective 自由分析
        """
        
        # 构建 SQL 查询（返回明细数据，不聚合）
        query = text("""
        SELECT 
            -- 客户维度
            CASE 
                WHEN c.customer_source = 'TRAFFIC' THEN '流量'
                WHEN c.customer_source = 'REFERRAL' THEN '推荐'
                ELSE c.customer_source::text
            END as customer_source,
            
            CASE 
                WHEN cg.group_type = 'FAMILY' THEN '家庭'
                WHEN cg.group_type = 'COMPANY' THEN '公司'
                WHEN cg.group_type = 'OTHER' THEN '其他'
                ELSE cg.group_type::text
            END as group_type,
            
            c.customer_nickname,
            
            -- 订单维度
            o.order_id,
            o.order_type,
            o.total_amount,
            o.product_quantity,
            o.shop,
            o.handler,
            
            -- 时间维度
            o.order_created_date,
            EXTRACT(YEAR FROM o.order_created_date) as year,
            TO_CHAR(o.order_created_date, 'YYYY-MM') as month,
            TO_CHAR(o.order_created_date, 'YYYY-Q') as quarter,
            EXTRACT(DOW FROM o.order_created_date) as day_of_week
            
        FROM customers c
        LEFT JOIN customer_groups cg ON c.group_id = cg.group_id
        INNER JOIN orders o ON c.customer_nickname = o.customer_nickname
        WHERE o.order_created_date >= :start_date 
          AND o.order_created_date <= :end_date
          AND o.total_amount IS NOT NULL
        ORDER BY o.order_created_date
        """)
        
        # 执行查询
        result = await db.execute(
            query,
            {
                "start_date": request.date_range.start,
                "end_date": request.date_range.end
            }
        )
        
        rows = result.fetchall()
        
        # 转换为 Polars DataFrame
        df = pl.DataFrame({
            "customer_source": [row[0] for row in rows],
            "group_type": [row[1] for row in rows],
            "customer_nickname": [row[2] for row in rows],
            "order_id": [row[3] for row in rows],
            "order_type": [row[4] for row in rows],
            "total_amount": [float(row[5]) for row in rows],
            "product_quantity": [int(row[6]) for row in rows],
            "shop": [row[7] for row in rows],
            "handler": [row[8] for row in rows],
            "order_created_date": [row[9] for row in rows],
            "year": [int(row[10]) for row in rows],
            "month": [row[11] for row in rows],
            "quarter": [row[12] for row in rows],
            "day_of_week": [int(row[13]) for row in rows],
        })
        
        return df
```

### 2. 前端：Perspective 4.0 集成

#### 主组件设计

```vue
<!-- OneManage_web/src/views/business-analysis/index.vue -->
<template>
  <div class="business-analysis-container">
    <!-- 顶部：简化的配置面板 -->
    <n-card title="数据加载" class="config-panel">
      <n-space>
        <!-- 只需要选择时间范围 -->
        <n-date-picker
          v-model:value="dateRange"
          type="daterange"
          :default-value="defaultDateRange"
        />
        
        <n-button 
          type="primary" 
          @click="loadData" 
          :loading="loading"
        >
          加载数据
        </n-button>
        
        <n-tag v-if="dataLoaded" type="success">
          已加载 {{ rowCount }} 条数据
        </n-tag>
      </n-space>
    </n-card>

    <!-- 中部：Perspective Viewer -->
    <n-card title="数据分析" class="perspective-panel">
      <perspective-viewer
        ref="viewerRef"
        :editable="false"
        :settings="true"
        theme="Pro Light"
      />
    </n-card>

    <!-- 底部：ECharts 高级图表（可选） -->
    <n-card 
      v-if="showECharts" 
      title="高级图表" 
      class="echarts-panel"
    >
      <div ref="echartRef" style="height: 400px"></div>
      
      <n-space style="margin-top: 16px">
        <n-button @click="syncToECharts">
          同步到 ECharts
        </n-button>
        <n-switch v-model:value="autoSync">
          <template #checked>自动同步</template>
          <template #unchecked>手动同步</template>
        </n-switch>
      </n-space>
    </n-card>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, watch } from 'vue'
import { NCard, NSpace, NDatePicker, NButton, NTag, NSwitch, useMessage } from 'naive-ui'
import perspective from '@finos/perspective'
import '@finos/perspective-viewer'
import '@finos/perspective-viewer-datagrid'
import '@finos/perspective-viewer-d3fc'
import * as echarts from 'echarts'

const message = useMessage()
const viewerRef = ref<any>(null)
const echartRef = ref<HTMLElement>()
const dateRange = ref<[number, number]>()
const loading = ref(false)
const dataLoaded = ref(false)
const rowCount = ref(0)
const showECharts = ref(false)
const autoSync = ref(false)

let perspectiveTable: any = null
let echartInstance: echarts.ECharts | null = null

// 默认日期范围（最近一年）
const getDefaultDateRange = (): [number, number] => {
  const end = new Date()
  const start = new Date()
  start.setFullYear(start.getFullYear() - 1)
  return [start.getTime(), end.getTime()]
}

const defaultDateRange = getDefaultDateRange()
dateRange.value = defaultDateRange

/**
 * 加载数据
 */
const loadData = async () => {
  if (!dateRange.value) {
    message.error('请选择时间范围')
    return
  }

  loading.value = true

  try {
    const [startTime, endTime] = dateRange.value
    const start = new Date(startTime)
    const end = new Date(endTime)

    // 格式化日期
    const formatDate = (date: Date) => {
      const year = date.getFullYear()
      const month = String(date.getMonth() + 1).padStart(2, '0')
      const day = String(date.getDate()).padStart(2, '0')
      return `${year}-${month}-${day}`
    }

    // 请求 Arrow 数据
    const response = await fetch('http://localhost:8000/api/analytics/customer-orders/arrow', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        date_range: {
          start: formatDate(start),
          end: formatDate(end)
        }
      })
    })

    if (!response.ok) {
      throw new Error('数据加载失败')
    }

    // 获取 Arrow 二进制数据
    const arrowBuffer = await response.arrayBuffer()

    // 创建 Perspective 表
    const worker = perspective.worker()
    perspectiveTable = await worker.table(arrowBuffer)

    // 获取行数
    const size = await perspectiveTable.size()
    rowCount.value = size

    // 加载到 Viewer
    await viewerRef.value.load(perspectiveTable)

    // 配置默认视图
    await viewerRef.value.restore({
      plugin: 'Datagrid',
      group_by: ['customer_source'],
      columns: ['order_id', 'total_amount', 'product_quantity'],
      aggregates: {
        order_id: 'distinct count',
        total_amount: 'sum',
        product_quantity: 'sum'
      },
      sort: [['total_amount', 'desc']]
    })

    dataLoaded.value = true
    message.success(`数据加载成功，共 ${rowCount.value} 条记录`)

  } catch (error: any) {
    console.error('数据加载失败:', error)
    message.error(error.message || '数据加载失败')
  } finally {
    loading.value = false
  }
}

/**
 * 同步到 ECharts
 */
const syncToECharts = async () => {
  if (!perspectiveTable) {
    message.warning('请先加载数据')
    return
  }

  try {
    // 从 Perspective 获取当前视图数据
    const view = await viewerRef.value.getView()
    const data = await view.to_json()

    // 初始化 ECharts
    if (!echartInstance && echartRef.value) {
      echartInstance = echarts.init(echartRef.value)
    }

    // 配置 ECharts
    const option = {
      tooltip: {
        trigger: 'axis',
        axisPointer: { type: 'shadow' }
      },
      legend: {
        data: ['订单数量', '累计金额']
      },
      xAxis: {
        type: 'category',
        data: data.map((d: any) => d.customer_source || d.group_type)
      },
      yAxis: [
        {
          type: 'value',
          name: '订单数量'
        },
        {
          type: 'value',
          name: '累计金额',
          position: 'right'
        }
      ],
      series: [
        {
          name: '订单数量',
          type: 'bar',
          data: data.map((d: any) => d.order_id)
        },
        {
          name: '累计金额',
          type: 'bar',
          yAxisIndex: 1,
          data: data.map((d: any) => d.total_amount)
        }
      ]
    }

    echartInstance?.setOption(option)
    showECharts.value = true
    message.success('ECharts 同步成功')

  } catch (error: any) {
    console.error('ECharts 同步失败:', error)
    message.error('ECharts 同步失败')
  }
}

// 监听 Perspective 视图变化，自动同步到 ECharts
watch(autoSync, (enabled) => {
  if (enabled && viewerRef.value) {
    viewerRef.value.addEventListener('perspective-view-update', syncToECharts)
  } else if (viewerRef.value) {
    viewerRef.value.removeEventListener('perspective-view-update', syncToECharts)
  }
})

onMounted(() => {
  // 自动加载数据
  loadData()
})
</script>

<style scoped>
.business-analysis-container {
  padding: 24px;
  background-color: #f5f7fa;
  min-height: 100vh;
}

.config-panel {
  margin-bottom: 20px;
}

.perspective-panel {
  margin-bottom: 20px;
}

perspective-viewer {
  height: 600px;
  display: block;
}

.echarts-panel {
  margin-bottom: 20px;
}
</style>
```

### 3. HTTP 缓存策略

```python
# features/business_analysis/routes.py

import hashlib
from fastapi import Header

def generate_etag(data: bytes) -> str:
    """生成 ETag（基于数据内容的 MD5 哈希）"""
    return hashlib.md5(data).hexdigest()

@router.post("/customer-orders/arrow")
async def analyze_customer_orders_arrow(
    request: CustomerOrderAnalysisRequest,
    db: AsyncSession = Depends(get_db),
    if_none_match: str | None = Header(None)
) -> Response:
    """
    返回 Arrow IPC 格式的分析数据
    
    使用 HTTP 缓存机制：
    1. 生成 ETag（基于数据内容）
    2. 检查客户端的 If-None-Match 头
    3. 如果匹配，返回 304 Not Modified
    4. 否则返回完整数据，带 ETag 和 Cache-Control 头
    """
    
    # 查询数据并转换为 Arrow
    df = await customer_order_service.get_raw_data(db, request)
    arrow_bytes = df_to_arrow_bytes(df)
    
    # 生成 ETag
    etag = f'"{generate_etag(arrow_bytes)}"'
    
    # 检查缓存
    if if_none_match == etag:
        return Response(status_code=304, headers={"ETag": etag})
    
    # 返回数据
    return Response(
        content=arrow_bytes,
        media_type="application/vnd.apache.arrow.stream",
        headers={
            "ETag": etag,
            "Cache-Control": "private, max-age=3600"
        }
    )
```

**HTTP 缓存优势**：
- ✅ 无需额外的 Redis 服务
- ✅ 浏览器自动管理缓存
- ✅ 支持条件请求（304 Not Modified）
- ✅ 减少网络传输（缓存命中时只传输头部）

## 性能优化策略

### 1. 两层加速

```
第一层: 浏览器缓存 (HTTP Cache)
- 响应时间: 0ms（无网络请求）
- 使用 ETag 和 Cache-Control
- 缓存时间: 1 小时

第二层: 数据库查询 + 索引优化
- 响应时间: 1-2秒（30W 行数据）
- 使用已有的数据库索引
- 支持任意时间范围和筛选条件
```

### 2. 前端优化

```javascript
// 虚拟滚动（Perspective 内置）
// 支持 1000W 行数据流畅滚动

// WebAssembly 加速
// Perspective 使用 WASM 进行高性能计算

// 增量更新
// 只更新变化的部分，不重新渲染整个表格
```

## 技术栈版本

### 后端
- Python: 3.12
- FastAPI: 0.115+
- Polars: 1.27+
- PyArrow: 20.0+

### 前端
- Vue: 3.5+
- Perspective: 4.0+ (`@finos/perspective`, `@finos/perspective-viewer`)
- ECharts: 6.0+
- Naive UI: 2.41+

## 扩展性设计

### 未来扩展方向

1. **性能优化**
   - 当数据量增长到 100W+ 时，考虑引入物化视图
   - 使用 Redis 缓存热点数据
   - 实现查询结果的服务端缓存

2. **实时数据流**
   - 使用 WebSocket 推送增量数据
   - Perspective 支持 streaming update

3. **协作功能**
   - 保存和分享分析视图
   - 多用户同时分析

4. **自定义计算字段**
   - 前端公式编辑器
   - 动态计算新指标

5. **AI 驱动洞察**
   - 自动发现数据趋势
   - 异常检测和预警
