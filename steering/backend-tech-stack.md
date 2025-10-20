---
inclusion: fileMatch
fileMatchPattern: 'OneManage/**/*'
---

# 后端技术栈规范

## 语言和交流规范
- **所有对话**: 必须使用简体中文
- **代码注释**: 必须使用简体中文
- **文档编写**: 必须使用简体中文
- **变量函数名**: 使用英文，但注释用简体中文

## 核心技术栈

### Python 环境
- **Python 版本**: 3.12.* (严格版本控制)
- **包管理器**: PDM (项目依赖管理)
- **镜像源**: 阿里云 PyPI 镜像 (https://mirrors.aliyun.com/pypi/simple/)

### Web 框架
- **后端框架**: FastAPI (>=0.115.12)
- **ASGI 服务器**: Uvicorn
- **API 文档**: 自动生成 OpenAPI/Swagger
- **架构模式**: Feature-Driven 模块化架构
- **序列化**: msgspec (高性能) + Pydantic (复杂场景)
- **日志系统**: loguru (结构化日志)

### 数据库和 ORM
- **数据库**: PostgreSQL 17
- **ORM**: SQLModel (>=0.0.24)
- **数据库迁移**: Alembic (>=1.16.1)
- **连接池**: AsyncPG (异步连接)
- **类型定义**: 使用现代联合类型语法 (`| None`)

### 异步任务和消息队列
- **任务队列**: TaskIQ (>=0.11.17)
- **消息中间件**: NATS 2.12 + JetStream
- **架构模式**: 事件驱动 + 发布订阅
- **任务超时**: 300秒 (5分钟)
- **重试次数**: 最多3次
- **同步阈值**: 2秒以内优先同步处理

#### TaskIQ + NATS 架构图

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   FastAPI       │    │   NATS JetStream │    │   TaskIQ Worker │
│   (API层)       │    │   (消息代理)      │    │   (任务执行)     │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         │                        │                        │
         │ 1. API调用              │                        │
         ▼                        │                        │
┌─────────────────┐              │                        │
│  Router Handler │              │                        │
│  触发任务       │──────────────▶│ 2. 任务入队             │
└─────────────────┘              │                        │
                                 │                        │
                                 │ 3. 任务分发             ▼
                                 │                ┌─────────────────┐
                                 │                │  @broker.task   │
                                 │                │  执行业务逻辑    │
                                 │                └─────────────────┘
                                 │                        │
                                 │ 5. 事件发布             │ 4. 发布事件
                                 │ ◀──────────────────────┘
                                 │
                                 │ 6. 事件分发
                                 ▼
                        ┌─────────────────┐
                        │ @subscribe_event│
                        │ 事件监听器       │
                        └─────────────────┘
```

#### 核心组件配置

##### 1. Broker 配置 (`core/broker.py`)
```python
from taskiq_nats import PushBasedJetStreamBroker
from nats.js.api import ConsumerConfig, AckPolicy, DeliverPolicy

broker = PushBasedJetStreamBroker(
    servers=settings.nats_servers,
    stream_name=settings.NATS_STREAM,
    queue="taskiq",
    consumer_config=ConsumerConfig(
        name="taskiq",
        durable_name="taskiq",
        ack_wait=300,           # 5分钟任务超时
        max_deliver=3,          # 最多重试3次
        max_ack_pending=100,    # 并发处理100个任务
        ack_policy=AckPolicy.EXPLICIT,
        deliver_policy=DeliverPolicy.ALL,
    ),
    **nats_kwargs,
)
```

##### 2. 任务定义规范
```python
# features/*/tasks.py
@broker.task
async def async_process_data(data: dict, task_id: str) -> dict:
    """异步数据处理任务
    
    Args:
        data: 处理数据
        task_id: 任务ID
        
    Returns:
        dict: 处理结果
    """
    try:
        # 发布开始事件
        await publish_event(PROCESS_STARTED, {
            "task_id": task_id,
            "status": "started"
        })
        
        # 执行业务逻辑
        result = await do_business_logic(data)
        
        # 发布完成事件
        await publish_event(PROCESS_COMPLETED, {
            "task_id": task_id,
            "result": result,
            "status": "completed"
        })
        
        return result
        
    except Exception as e:
        # 发布失败事件
        await publish_event(PROCESS_FAILED, {
            "task_id": task_id,
            "error": str(e),
            "status": "failed"
        })
        raise
```

##### 3. 事件监听规范
```python
# features/*/tasks.py
@subscribe_event(PROCESS_COMPLETED, model=ProcessCompletedEvent)
async def on_process_completed(evt: ProcessCompletedEvent) -> None:
    """监听处理完成事件
    
    Args:
        evt: 事件数据模型
    """
    try:
        # 更新数据库状态
        async with get_session_context() as db:
            await update_task_status(db, evt.task_id, "completed")
            await db.commit()
            
        logger.info("任务完成", task_id=evt.task_id)
        
    except Exception as e:
        logger.error("事件处理失败", error=str(e))
```

#### 消息流转过程
1. **API触发**: 用户调用API接口
2. **任务入队**: Router将任务推送到NATS JetStream
3. **任务执行**: Worker从队列获取任务并执行
4. **事件发布**: 任务完成后发布事件到NATS
5. **事件监听**: 其他模块监听事件并执行后续处理

#### 模块间通信示例
```
PSD上传 → 解析任务 → 解析完成事件 → 背景生成任务
   │         │           │              │
   API    tasks.py    events/       tasks.py
```

#### TTL 机制应用
- **临时数据**: 图层预览URL设置24小时TTL
- **缓存管理**: 自动清理过期的预览文件
- **会话管理**: 用户会话自动过期

### 数据处理
- **数据分析库**: Polars (>=1.27.1) - 主要数据处理引擎
- **Excel 读取**: FastExcel (>=0.14.0) - 与 Polars 深度集成
- **数据连接**: ConnectorX (>=0.4.3) - 高性能数据库连接
- **Arrow 格式**: PyArrow (>=20.0.0) - 内存格式支持

#### 数据读取和写入规范
```python
import polars as pl
from fastexcel import read_excel
import connectorx as cx
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text, create_engine

# Excel/CSV 文件读取 - 使用 FastExcel
df_excel = pl.from_pandas(read_excel("file.xlsx").to_pandas())
df_csv = pl.read_csv("file.csv")

# 数据库读取 - 使用 ConnectorX (高性能)
df_db = pl.from_pandas(cx.read_sql(
    "SELECT * FROM orders", 
    settings.database_url_sync
))

# Polars 写入数据库 - 多种方案
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
        import tempfile
        import os
        
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

### 图像处理
- **图像库**: Pillow (>=11.3.0)
- **计算机视觉**: OpenCV (>=4.12.0.88)
- **PSD 处理**: psd-tools (>=1.10.9)

### 云服务和存储
- **对象存储**: Cloudflare R2 (主要) / 阿里云 OSS (备选)
- **存储驱动**: boto3 (S3兼容接口，便于切换)
- **弹性计算**: 阿里云 ECI (计算密集型任务)
- **触发方式**: NATS 事件触发 ECI 容器

#### 对象存储配置规范
```python
# 使用 boto3 S3兼容接口，支持多种对象存储
import boto3
from botocore.config import Config

# Cloudflare R2 配置
s3_client = boto3.client(
    's3',
    endpoint_url=settings.S3_ENDPOINT,  # R2: https://account-id.r2.cloudflarestorage.com
    aws_access_key_id=settings.S3_ACCESS_KEY_ID,
    aws_secret_access_key=settings.S3_SECRET_ACCESS_KEY,
    region_name=settings.S3_REGION,  # R2: "auto"
    config=Config(signature_version='s3v4')
)

# 阿里云 OSS 配置 (切换时只需修改 endpoint)
# endpoint_url: https://oss-cn-hangzhou.aliyuncs.com
```

## 禁用技术

### 明确不使用的技术
- ❌ **Redis**: 不设计 Redis 缓存方案
- ❌ **Celery**: 使用 TaskIQ 替代
- ❌ **Pandas**: 优先使用 Polars
- ❌ **Kubernetes**: 只使用 Docker Compose

## 性能策略

### 同步 vs 异步决策
- **2秒以内**: 优先同步处理
- **计算密集型**: 使用 TaskIQ + ECI 弹性伸缩
- **IO 密集型**: 使用 AsyncPG + 异步处理

### 扩展策略
- **水平扩展**: Docker Compose 多实例
- **弹性计算**: 阿里云 ECI 按需启动
- **事件驱动**: NATS JetStream 解耦服务

## 开发工具

### 代码质量
- **格式化**: Black (line-length=88)
- **导入排序**: isort (profile="black")
- **类型检查**: MyPy (Python 3.12)
- **测试框架**: pytest

### 开发环境
- **调试工具**: ipdb
- **交互式**: IPython
- **依赖检查**: deptry