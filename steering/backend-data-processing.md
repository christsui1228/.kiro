---
inclusion: fileMatch
fileMatchPattern: '**/crud.py,**/services.py,**/tasks.py'
---

# 数据处理规范

## Polars 数据处理规范

### 写入数据库的真实情况
Polars 的 `write_database()` 方法底层实现：
1. 将 Polars DataFrame 转换为 pandas DataFrame  
2. 使用 pandas 的 `to_sql()` 方法
3. 依赖 SQLAlchemy 引擎

### 写入策略选择
根据数据量和性能需求选择：

1. **小数据量 (<10K行)**: `df.write_database()` - 简单直接
2. **中等数据量 (10K-100K行)**: 手动分批 + pandas.to_sql - 控制内存
3. **大数据量 (>100K行)**: CSV + PostgreSQL COPY - 最高性能

```python
import polars as pl
from datetime import datetime, timezone
from sqlalchemy import create_engine, text
from sqlalchemy.ext.asyncio import AsyncSession
import tempfile
import os

async def write_polars_to_db(df: pl.DataFrame, table_name: str):
    """
    Polars 写入数据库的方案选择
    
    方案1: Polars 原生 write_database (推荐小数据量)
    方案2: 手动转 pandas + 分批处理 (推荐中等数据量)  
    方案3: CSV + COPY 命令 (推荐大数据量)
    """
    
    # 方案1: Polars 原生方法 (底层仍然是 pandas.to_sql)
    if len(df) < 10000:
        # 注意：write_database 底层会转换为 pandas 并使用 to_sql
        df.write_database(
            table_name=table_name,
            connection=settings.database_url_sync,
            if_table_exists="append"
        )
    
    # 方案2: 手动控制 pandas 转换和分批写入
    elif len(df) < 100000:
        batch_size = 5000
        engine = create_engine(settings.database_url_sync)
        
        # 分批转换和写入，控制内存使用
        for i in range(0, len(df), batch_size):
            batch_df = df.slice(i, batch_size)
            pandas_batch = batch_df.to_pandas()
            
            pandas_batch.to_sql(
                table_name, 
                engine, 
                if_exists='append', 
                index=False,
                method='multi'  # 批量插入优化
            )
    
    # 方案3: 超大数据量使用 PostgreSQL COPY (最高性能)
    else:
        # 导出为临时 CSV
        with tempfile.NamedTemporaryFile(mode='w', suffix='.csv', delete=False) as f:
            temp_csv = f.name
            
        df.write_csv(temp_csv)
        
        # 使用异步 COPY 命令
        async with AsyncSession() as session:
            copy_sql = text(f"""
            COPY {table_name} FROM '{temp_csv}' 
            WITH (FORMAT CSV, HEADER TRUE)
            """)
            await session.execute(copy_sql)
            await session.commit()
        
        # 清理临时文件
        os.unlink(temp_csv)
```

### 数据处理示例

```python
async def import_orders(df: pl.DataFrame):
    """导入订单数据 - 展示 Polars 数据处理能力"""
    
    # Polars 的强项：高性能数据清洗和转换
    cleaned_df = (
        df
        .with_columns([
            # 时间转换
            pl.col("order_date").str.to_datetime().alias("order_date"),
            # 数值转换和验证
            pl.col("amount").cast(pl.Float64),
            # 添加时间戳
            pl.lit(datetime.now(timezone.utc)).alias("created_at"),
            pl.lit(datetime.now(timezone.utc)).alias("updated_at"),
            # 字符串清理
            pl.col("customer_name").str.strip_chars().str.to_uppercase()
        ])
        .filter(
            # 数据验证
            (pl.col("amount") > 0) & 
            (pl.col("order_date").is_not_null()) &
            (pl.col("customer_name").str.len_chars() > 0)
        )
        .drop_nulls(subset=["order_id"])
        .unique(subset=["order_id"])  # 去重
    )
    
    # 写入数据库 (根据数据量自动选择策略)
    await write_polars_to_db(cleaned_df, "orders")
    
    return {
        "total_rows": len(df),
        "cleaned_rows": len(cleaned_df),
        "success_rate": len(cleaned_df) / len(df) * 100
    }
```

### Polars 常用操作

```python
import polars as pl

# 读取数据
df_csv = pl.read_csv("file.csv")
df_excel = pl.from_pandas(read_excel("file.xlsx").to_pandas())

# 使用 ConnectorX 高性能读取数据库
import connectorx as cx
df_db = pl.from_pandas(cx.read_sql(
    "SELECT * FROM orders", 
    settings.database_url_sync
))

# 数据转换
result = (
    df
    .select([
        pl.col("user_id"),
        pl.col("amount").sum().alias("total_amount"),
        pl.col("order_date").max().alias("last_order")
    ])
    .group_by("user_id")
    .agg([
        pl.col("total_amount").sum(),
        pl.col("last_order").max()
    ])
    .filter(pl.col("total_amount") > 1000)
    .sort("total_amount", descending=True)
)

# 数据合并
merged = df1.join(df2, on="user_id", how="left")

# 数据透视
pivot = df.pivot(
    values="amount",
    index="user_id",
    columns="product_category"
)
```

## Polars vs Pandas 使用场景

### Polars 优势场景
- ✅ 大数据量处理（>100MB）
- ✅ 数据清洗和转换
- ✅ 聚合分析和统计
- ✅ 多列操作和链式调用
- ✅ 内存效率要求高

### Pandas 优势场景
- ✅ 数据库写入（to_sql）
- ✅ 生态兼容性（sklearn、matplotlib）
- ✅ 复杂的时间序列操作
- ✅ 小数据量快速原型

### 最佳实践
**Polars 处理 + pandas 写入**

```python
# 使用 Polars 进行数据处理
cleaned_df = (
    pl.read_csv("large_file.csv")
    .filter(pl.col("status") == "active")
    .with_columns([
        pl.col("amount").cast(pl.Float64),
        pl.col("date").str.to_datetime()
    ])
)

# 转换为 pandas 写入数据库
pandas_df = cleaned_df.to_pandas()
pandas_df.to_sql("table_name", engine, if_exists="append", index=False)
```

## 数据验证规范

### 使用 Polars 进行数据验证

```python
def validate_dataframe(df: pl.DataFrame) -> tuple[pl.DataFrame, list[str]]:
    """验证数据框并返回清洗后的数据和错误信息"""
    errors = []
    
    # 检查必填字段
    required_columns = ["user_id", "amount", "order_date"]
    missing_columns = set(required_columns) - set(df.columns)
    if missing_columns:
        errors.append(f"缺少必填列: {missing_columns}")
        return df, errors
    
    # 数据类型验证
    try:
        validated_df = df.with_columns([
            pl.col("amount").cast(pl.Float64),
            pl.col("order_date").str.to_datetime()
        ])
    except Exception as e:
        errors.append(f"数据类型转换失败: {e}")
        return df, errors
    
    # 业务规则验证
    invalid_rows = validated_df.filter(
        (pl.col("amount") <= 0) | 
        (pl.col("order_date").is_null())
    )
    
    if len(invalid_rows) > 0:
        errors.append(f"发现 {len(invalid_rows)} 行无效数据")
    
    # 返回有效数据
    valid_df = validated_df.filter(
        (pl.col("amount") > 0) & 
        (pl.col("order_date").is_not_null())
    )
    
    return valid_df, errors
```

## 性能优化建议

### 内存优化
```python
# 使用 lazy evaluation
lazy_df = pl.scan_csv("large_file.csv")
result = (
    lazy_df
    .filter(pl.col("status") == "active")
    .select(["user_id", "amount"])
    .collect()  # 只在最后执行
)

# 分块处理大文件
chunk_size = 100000
for chunk in pl.read_csv_batched("huge_file.csv", batch_size=chunk_size):
    process_chunk(chunk)
```

### 并行处理
```python
# Polars 自动并行处理
df = pl.read_csv("file.csv", n_threads=4)

# 使用 TaskIQ 并行处理多个文件
@broker.task
async def process_file(file_path: str):
    df = pl.read_csv(file_path)
    # 处理逻辑
    return result

# 并行提交多个任务
tasks = [process_file.kiq(file) for file in file_list]
results = await asyncio.gather(*tasks)
```
