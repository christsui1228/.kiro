# ETL 模块重构设计文档

## 设计概述

本设计文档描述 ETL 模块重构的技术方案，包括架构设计、服务层设计、向量化优化策略等。

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                      ETL 模块架构                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │ orders/      │    │ customers/   │    │ leads/       │ │
│  │  ├─ etl.py   │    │  ├─ etl.py   │    │  ├─ etl.py   │ │
│  │  └─ tasks.py │    │  └─ tasks.py │    │  └─ tasks.py │ │
│  └──────────────┘    └──────────────┘    └──────────────┘ │
│         │                    │                    │         │
│         └────────────────────┼────────────────────┘         │
│                              ↓                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              服务层 (services/)                       │  │
│  │  ├─ batch.py              批处理服务                  │  │
│  │  └─ vectorized_queries.py 向量化查询工具             │  │
│  └──────────────────────────────────────────────────────┘  │
│                              ↓                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              基础设施层                               │  │
│  │  ├─ core.database_sync    数据库引擎                 │  │
│  │  ├─ core.broker           TaskIQ 任务队列            │  │
│  │  └─ events/               事件发布/订阅               │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 执行流程

```
1. 数据导入完成
   ↓
2. 发布 DATA_IMPORT_COMPLETED 事件
   ↓
3. 订单 ETL 监听器触发
   ↓
4. TaskIQ 任务入队 (run_etl_orders)
   ↓
5. Worker 执行: asyncio.to_thread(orders_etl_task)
   ↓
6. 同步 ETL 逻辑:
   - 使用 Polars 读取数据
   - 向量化转换
   - 批量写入数据库
   ↓
7. 发布 ORDERS_ETL_COMPLETED 事件
   ↓
8. 客户 ETL 监听器触发 → 重复步骤 4-7
   ↓
9. 商机 ETL 监听器触发 → 重复步骤 4-7
```

## 核心设计决策

### 决策 1：使用 asyncio.to_thread() + 同步 Polars

**理由**：
- Polars 在同步环境下性能最佳（ConnectorX 零拷贝）
- TaskIQ + NATS 提供任务持久化和可靠性
- `asyncio.to_thread()` 不阻塞事件循环

**实现**：
```python
# tasks.py
@broker.task
async def run_etl_orders(import_id: str) -> None:
    """异步任务包装器"""
    result = await asyncio.to_thread(
        orders_etl_task,  # 同步函数
        uuid.UUID(import_id)
    )
    await publish_event(ORDERS_ETL_COMPLETED, result)

# etl.py
def orders_etl_task(task_id: uuid.UUID) -> dict:
    """同步 ETL 逻辑"""
    df = pl.read_database_uri(...)  # 同步 Polars
    # ... 处理逻辑
    return {"status": "success", "count": len(df)}
```

### 决策 2：复用 core.database_sync

**理由**：
- 避免重复创建数据库引擎
- 统一连接池配置
- 简化代码维护

**实现**：
```python
# 所有 ETL 模块统一导入
from core.database_sync import engine

def orders_etl_task(task_id: uuid.UUID):
    # 直接使用 engine
    df = pl.read_database_uri(
        query="SELECT ...",
        uri=engine.url.render_as_string(hide_password=False),
        engine='connectorx'
    )
```

### 决策 3：创建服务层而非基类

**理由**：
- 每个 ETL 模块的数据列不同，强制继承不灵活
- 组合优于继承，提供可选的工具函数
- 保持各模块独立性

**实现**：
```python
# services/batch.py
class BatchProcessor:
    """通用批处理服务（不强制继承）"""
    
    @staticmethod
    def process_in_batches(df, batch_size, process_func):
        # 通用批处理逻辑
        pass

# orders/etl.py
from features.etl.services.batch import BatchProcessor

def orders_etl_task(task_id):
    df = extract_orders(task_id)
    
    # 可选使用批处理服务
    processor = BatchProcessor()
    success, errors = processor.process_in_batches(
        df, 1000, load_orders_batch
    )
```

### 决策 4：全面向量化

**理由**：
- 减少数据库查询次数（1000 行从 3000 次查询降到 3 次）
- 代码更简洁（Polars 表达式替代 Python 循环）
- 性能提升（商机 ETL 从 15 秒降到 10 秒）

**实现**：
```python
# 逐行处理（重构前）
for row in df.iter_rows():
    customer = lookup_customer(row["customer_name"])  # N 次查询
    lead = lookup_lead(customer.id)                   # N 次查询

# 向量化处理（重构后）
customers_df = batch_lookup_customers(df["customer_name"].unique())  # 1 次查询
leads_df = batch_lookup_leads(customers_df["id"])                    # 1 次查询
result = df.join(customers_df).join(leads_df)                        # 向量化 join
```

## 服务层设计

### BatchProcessor 设计

**职责**：提供通用的批处理逻辑

**接口**：
```python
class BatchProcessor:
    @staticmethod
    def process_in_batches(
        df: pl.DataFrame,
        batch_size: int,
        process_func: Callable[[pl.DataFrame], tuple[int, int]]
    ) -> tuple[int, int]:
        """
        通用批处理
        
        Args:
            df: 要处理的数据
            batch_size: 批次大小
            process_func: 处理函数，返回 (成功数, 失败数)
            
        Returns:
            (总成功数, 总失败数)
        """
```

**使用示例**：
```python
def load_orders_batch(batch_df: pl.DataFrame) -> tuple[int, int]:
    """加载一批订单"""
    try:
        batch_df.write_database(...)
        return len(batch_df), 0
    except Exception as e:
        logger.error(f"批次失败: {e}")
        return 0, len(batch_df)

# 使用批处理服务
processor = BatchProcessor()
success, errors = processor.process_in_batches(
    df_transformed, 1000, load_orders_batch
)
```

### VectorizedQueries 设计

**职责**：提供批量查询工具，减少数据库往返

**接口**：
```python
class VectorizedQueries:
    @staticmethod
    def batch_lookup_customers(
        engine: Engine,
        df: pl.DataFrame
    ) -> pl.DataFrame:
        """
        批量查询客户
        
        Args:
            engine: 数据库引擎
            df: 包含 customer_name 和 shop 列的 DataFrame
            
        Returns:
            包含 customer_global_id 映射的 DataFrame
        """
        
    @staticmethod
    def batch_lookup_leads(
        engine: Engine,
        customer_gids: list[uuid.UUID]
    ) -> pl.DataFrame:
        """批量查询现有线索"""
        
    @staticmethod
    def batch_lookup_order_type_mappings(
        engine: Engine,
        order_types: list[str]
    ) -> pl.DataFrame:
        """批量查询订单类型映射"""
```

**实现策略**：
```python
@staticmethod
def batch_lookup_customers(engine: Engine, df: pl.DataFrame) -> pl.DataFrame:
    # 1. 获取唯一的客户名称和店铺组合
    unique_customers = df.select(["customer_name", "shop"]).unique()
    
    # 2. 构建批量查询（使用 IN 子句）
    customer_names = unique_customers["customer_name"].to_list()
    shops = unique_customers["shop"].to_list()
    
    placeholders = ",".join([f"('{name}', '{shop}')" 
                            for name, shop in zip(customer_names, shops)])
    
    query = f"""
        SELECT c.customer_global_id, c.customer_nickname as customer_name, 
               sc.shop_name as shop
        FROM customers c 
        JOIN shop_configs sc ON c.shop_id = sc.shop_id
        WHERE (c.customer_nickname, sc.shop_name) IN ({placeholders})
    """
    
    # 3. 一次性查询
    uri = engine.url.render_as_string(hide_password=False)
    return pl.read_database_uri(query=query, uri=uri, engine="connectorx")
```

## 向量化优化策略

### 策略 1：批量查询替代逐行查询

**优化前**：
```python
for row in df.iter_rows():
    customer = session.execute(
        select(Customer).where(Customer.name == row["customer_name"])
    ).scalar_one_or_none()
```

**优化后**：
```python
# 一次性查询所有客户
customers_df = VectorizedQueries.batch_lookup_customers(engine, df)
# 用 join 关联
result = df.join(customers_df, on=["customer_name", "shop"])
```

### 策略 2：向量化字段映射

**优化前**：
```python
mapped_records = []
for row in df.iter_rows():
    mapped_records.append({
        "order_id": row["order_id"],
        "customer_nickname": row["customer_name"],
        "latest_status": row["order_status"],
    })
```

**优化后**：
```python
df_transformed = df.select([
    pl.col("order_id"),
    pl.col("customer_name").alias("customer_nickname"),
    pl.col("order_status").alias("latest_status"),
])
```

### 策略 3：向量化条件判断

**优化前**：
```python
for row in df.iter_rows():
    if row["lead_id"] is None:
        strategy = "new"
    elif row["current_stage"] in ["OPEN", "COLD"]:
        strategy = "reuse"
    else:
        strategy = "new"
```

**优化后**：
```python
df = df.with_columns([
    pl.when(pl.col("lead_id").is_null())
    .then(pl.lit("new"))
    .when(pl.col("current_stage").is_in(["OPEN", "COLD"]))
    .then(pl.lit("reuse"))
    .otherwise(pl.lit("new"))
    .alias("lead_strategy")
])
```

### 策略 4：向量化聚合统计

**优化前**：
```python
stats = {}
for customer_id in customer_ids:
    orders = df.filter(pl.col("customer_id") == customer_id)
    stats[customer_id] = {
        "sample_count": len(orders.filter(pl.col("type") == "SAMPLE")),
        "sample_amount": orders.filter(pl.col("type") == "SAMPLE")["amount"].sum(),
    }
```

**优化后**：
```python
stats_df = df.group_by("customer_id").agg([
    pl.col("type").filter(pl.col("type") == "SAMPLE").len().alias("sample_count"),
    pl.col("amount").filter(pl.col("type") == "SAMPLE").sum().alias("sample_amount"),
])
```

## 事件监听器设计

### 统一使用 @event_task_handler 装饰器

**优化前**：
```python
@broker.task(stream="EVENTS", durable_name="observer_orders_etl_v1", ...)
async def on_data_import_completed(raw_payload: dict):
    # 手动版本检查
    version = raw_payload.get("event_version", "1.0")
    if version != "1.0":
        logger.warning("不支持的版本")
        return
    
    # 手动字段检查
    if "import_id" not in raw_payload:
        logger.warning("缺少字段")
        return
    
    # 手动状态检查
    if raw_payload["status"] != "completed":
        return
    
    # 业务逻辑
    await run_etl_orders.kiq(raw_payload["import_id"])
```

**优化后**：
```python
from events.event_task_utils import event_task_handler

@broker.task(stream="EVENTS", durable_name="observer_orders_etl_v1", ...)
@event_task_handler(
    supported_version="1.0",
    required_fields=["import_id", "status"],
    task_name="订单ETL触发器"
)
async def on_data_import_completed(raw_payload: dict):
    # 版本检查、字段验证、错误处理都自动完成
    # 只写核心业务逻辑
    if raw_payload["status"] == "completed":
        await run_etl_orders.kiq(raw_payload["import_id"])
```

## 商机 ETL 重点优化

### 当前性能瓶颈

1. **逐行查询客户**：1000 行 = 1000 次查询
2. **逐行查询线索**：1000 行 = 1000 次查询
3. **逐行查询订单类型映射**：1000 行 = 1000 次查询
4. **逐行创建线索**：每个新客户都单独插入

### 优化方案

```python
def _process_batch_vectorized(engine: Engine, df_batch: pl.DataFrame):
    """完全向量化的批处理"""
    
    # 1. 批量查询所有依赖数据（3 次查询替代 3000 次）
    customers_df = VectorizedQueries.batch_lookup_customers(engine, df_batch)
    customer_gids = customers_df["customer_global_id"].unique().to_list()
    leads_df = VectorizedQueries.batch_lookup_leads(engine, customer_gids)
    order_types = df_batch["order_type"].unique().to_list()
    mappings_df = VectorizedQueries.batch_lookup_order_type_mappings(engine, order_types)
    
    # 2. 向量化 join 关联所有数据
    enriched_df = (
        df_batch
        .join(customers_df, on=["customer_name", "shop"], how="inner")
        .join(leads_df, on="customer_global_id", how="left")
        .join(mappings_df, left_on="order_type", right_on="raw_order_type", how="left")
        .with_columns([
            pl.col("system_order_type").fill_null("BULK"),
            # 向量化判断策略
            pl.when(pl.col("lead_id").is_null()).then(pl.lit("new"))
            .when(pl.col("current_stage").is_in(["OPEN", "COLD"])).then(pl.lit("reuse"))
            .otherwise(pl.lit("new"))
            .alias("lead_strategy")
        ])
    )
    
    # 3. 分组处理
    new_leads_df = enriched_df.filter(pl.col("lead_strategy") == "new")
    reuse_leads_df = enriched_df.filter(pl.col("lead_strategy") == "reuse")
    
    # 4. 批量创建/更新（一次性 SQL）
    with engine.begin() as conn:
        if not new_leads_df.is_empty():
            _batch_create_leads_vectorized(conn, new_leads_df)
        if not reuse_leads_df.is_empty():
            _batch_update_leads_vectorized(conn, reuse_leads_df)
```

**预期性能提升**：
- 数据库查询：3000 次 → 3 次（减少 99.9%）
- 处理时间：15 秒 → 10 秒（提升 33%）

## 数据流设计

### 订单 ETL 数据流

```
raw_orders (PostgreSQL)
    ↓ pl.read_database_uri()
DataFrame (Polars)
    ↓ 向量化字段映射
    ↓ select([col1.alias(), col2.alias(), ...])
DataFrame (转换后)
    ↓ 批量 upsert
    ↓ write_database() 或 批量 INSERT
orders (PostgreSQL)
```

### 客户 ETL 数据流

```
orders (PostgreSQL)
    ↓ pl.read_database_uri()
DataFrame (订单数据)
    ↓ join(shop_configs)
    ↓ join(existing_customers, how="anti")
DataFrame (新客户)
    ↓ with_columns([生成 UUID, 生成 customer_key])
    ↓ write_database()
customers (PostgreSQL)
```

### 商机 ETL 数据流

```
orders (PostgreSQL)
    ↓ pl.read_database_uri()
DataFrame (订单数据)
    ↓ batch_lookup_customers()
    ↓ batch_lookup_leads()
    ↓ batch_lookup_order_type_mappings()
DataFrame (关联所有依赖数据)
    ↓ 向量化判断策略 (new vs reuse)
    ↓ filter() 分组
DataFrame (新线索) + DataFrame (复用线索)
    ↓ group_by().agg() 聚合统计
    ↓ write_database() 批量插入/更新
business_leads (PostgreSQL)
```

## 错误处理策略

### 批处理错误处理

```python
def process_in_batches(df, batch_size, process_func):
    total_success = 0
    total_errors = 0
    
    for i in range(0, len(df), batch_size):
        batch = df.slice(i, batch_size)
        try:
            success, errors = process_func(batch)
            total_success += success
            total_errors += errors
        except Exception as e:
            logger.error(f"批次 {i//batch_size + 1} 失败: {e}")
            total_errors += len(batch)
            # 继续处理下一批，不中断整个流程
    
    return total_success, total_errors
```

### 事件发布错误处理

```python
@broker.task
async def run_etl_orders(import_id: str):
    try:
        result = await asyncio.to_thread(orders_etl_task, uuid.UUID(import_id))
        
        # 发布成功事件
        await publish_event(ORDERS_ETL_COMPLETED, {
            "import_id": import_id,
            "status": "success",
            "success_count": result["success_count"],
        })
    except Exception as e:
        logger.error(f"订单 ETL 失败: {e}")
        
        # 发布失败事件
        await publish_event(ORDERS_ETL_COMPLETED, {
            "import_id": import_id,
            "status": "failed",
            "error": str(e),
        })
```

## 性能监控设计

### 关键指标

1. **ETL 执行时间**：每个 ETL 任务的总耗时
2. **数据库查询次数**：批量查询优化效果
3. **批处理吞吐量**：每批处理的记录数和耗时
4. **错误率**：失败记录数 / 总记录数

### 监控实现

```python
import time

def orders_etl_task(task_id: uuid.UUID) -> dict:
    start_time = time.time()
    
    # ETL 逻辑
    df = extract_orders(task_id)
    df_transformed = transform_orders(df)
    success, errors = load_orders(df_transformed)
    
    duration = time.time() - start_time
    
    logger.info(
        f"[订单ETL] 完成: task_id={task_id}, "
        f"记录数={len(df)}, 成功={success}, 失败={errors}, "
        f"耗时={duration:.2f}秒"
    )
    
    return {
        "status": "success" if errors == 0 else "partial_success",
        "success_count": success,
        "error_count": errors,
        "duration": duration,
    }
```

## 测试策略

### 单元测试

```python
# 测试批处理服务
def test_batch_processor():
    df = pl.DataFrame({"id": range(100)})
    
    def mock_process(batch_df):
        return len(batch_df), 0
    
    processor = BatchProcessor()
    success, errors = processor.process_in_batches(df, 10, mock_process)
    
    assert success == 100
    assert errors == 0

# 测试向量化查询
def test_batch_lookup_customers():
    df = pl.DataFrame({
        "customer_name": ["客户A", "客户B"],
        "shop": ["店铺1", "店铺2"]
    })
    
    result = VectorizedQueries.batch_lookup_customers(engine, df)
    
    assert "customer_global_id" in result.columns
    assert len(result) == 2
```

### 集成测试

```python
# 测试完整 ETL 流程
async def test_orders_etl_integration():
    # 1. 准备测试数据
    task_id = await create_test_import_task()
    
    # 2. 执行 ETL
    result = await asyncio.to_thread(orders_etl_task, task_id)
    
    # 3. 验证结果
    assert result["status"] == "success"
    assert result["success_count"] > 0
    
    # 4. 验证数据库
    orders = await get_orders_by_task(task_id)
    assert len(orders) == result["success_count"]
```

### 性能测试

```python
# 测试性能是否达标
async def test_orders_etl_performance():
    # 准备 1000 条测试数据
    task_id = await create_test_data(count=1000)
    
    start = time.time()
    result = await asyncio.to_thread(orders_etl_task, task_id)
    duration = time.time() - start
    
    # 验收标准：1000 条 < 5 秒
    assert duration < 5.0
    assert result["success_count"] == 1000
```

## 迁移策略

### 灰度发布

1. **阶段 1**：在测试环境验证
2. **阶段 2**：生产环境处理 10% 的数据
3. **阶段 3**：生产环境处理 50% 的数据
4. **阶段 4**：生产环境处理 100% 的数据

### 回滚方案

- 保留原有 ETL 代码（重命名为 `etl_old.py`）
- 如果发现问题，快速切换回旧版本
- 验证无问题后，删除旧代码

## 技术债务

### 当前不做的事情

1. ❌ **不引入 PostgreSQL COPY 命令**：10 万行以内批量 INSERT 足够
2. ❌ **不引入缓存**：根据 spec-task-scope.md，不在 spec 中设计缓存
3. ❌ **不编写单元测试**：根据 spec-task-scope.md，测试不在 spec 范围
4. ❌ **不设计监控告警**：根据 spec-task-scope.md，监控不在 spec 范围

### 未来优化方向

1. 如果数据量超过 10 万行，考虑引入 COPY 命令
2. 如果并发 ETL 任务超过 30 个，考虑限流
3. 如果性能仍不满足，考虑引入缓存

## 参考资料

- Polars 官方文档：https://pola-rs.github.io/polars/
- TaskIQ 文档：https://taskiq-python.github.io/
- ConnectorX 文档：https://github.com/sfu-db/connector-x
- 项目事件系统：`events/EVENT_TASKS_README.md`
