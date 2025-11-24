# FastAPI Starter Template 设计文档

## 概述

本文档描述如何从 OneManage 项目中提取通用基础设施，创建一个独立的 FastAPI 项目模板。模板将位于 `~/coding/fastapi-starter-template/`，包含配置管理、数据库、消息队列、对象存储等核心组件。

## 设计目标

1. **最小化依赖**：只包含通用基础设施，不包含业务逻辑
2. **参数化配置**：所有配置项都可通过环境变量或初始化脚本替换
3. **开箱即用**：提供验证脚本，快速检查配置是否正确
4. **向后兼容**：保持与 OneManage 相同的技术栈和代码风格
5. **易于扩展**：提供示例模块，展示标准的代码结构

## 技术栈

- **Web 框架**: FastAPI 0.104+
- **数据库**: PostgreSQL + SQLModel + Alembic
- **消息队列**: NATS JetStream + TaskIQ
- **对象存储**: S3 兼容客户端（支持阿里云 OSS 和 Cloudflare R2）
- **日志**: Loguru
- **依赖管理**: PDM
- **代码质量**: Black + Ruff

## 项目结构


```
fastapi-starter-template/
├── .env.example                    # 配置模板
├── .gitignore                      # Git 忽略规则
├── pyproject.toml                  # PDM 依赖管理
├── pdm.lock                        # 依赖锁定文件
├── alembic.ini                     # 数据库迁移配置
├── main.py                         # FastAPI 应用入口
│
├── core/                           # 核心配置层
│   ├── __init__.py
│   ├── config.py                   # Pydantic Settings 配置管理
│   ├── database.py                 # 数据库连接和会话管理
│   └── broker.py                   # TaskIQ + NATS 消息队列配置
│
├── common/                         # 通用工具层
│   ├── __init__.py
│   ├── logging_config.py           # Loguru 日志配置
│   ├── response_envelope.py        # 统一响应格式
│   ├── envelope_middleware.py      # 响应包装中间件
│   ├── json_utils.py               # JSON 序列化工具
│   └── oss.py                      # S3 兼容存储客户端
│
├── events/                         # 事件驱动层
│   ├── __init__.py
│   ├── publisher.py                # 事件发布器
│   ├── schemas.py                  # 事件 Schema 定义
│   └── subjects.py                 # NATS 主题定义
│
├── scripts/                        # 工具脚本
│   ├── __init__.py
│   ├── init_project.py             # 项目初始化脚本
│   ├── test_db_connection.py       # 数据库连接测试
│   ├── test_oss_connection.py      # OSS 连接测试
│   ├── test_nats_connection.py     # NATS 连接测试
│   ├── health_check.py             # 一键健康检查
│   └── setup_jetstream.py          # NATS JetStream 初始化
│
├── migrations/                     # 数据库迁移
│   ├── env.py                      # Alembic 环境配置
│   ├── script.py.mako              # 迁移脚本模板
│   └── versions/                   # 迁移版本目录
│
└── features/                       # 业务模块
    └── example/                    # 示例模块
        ├── __init__.py
        ├── models.py               # SQLModel 数据模型
        ├── schemas.py              # Pydantic Schema
        ├── routers.py              # FastAPI 路由
        ├── services.py             # 业务逻辑
        └── tasks.py                # TaskIQ 异步任务
```


## 核心组件设计

### 1. 配置管理 (core/config.py)

**设计原则**：
- 使用 Pydantic Settings 管理所有配置
- 支持多环境配置文件（.env.development, .env.production）
- 通过环境变量 ENV 控制加载哪个配置文件
- 所有敏感信息通过环境变量注入

**从 OneManage 提取的内容**：
- `Settings` 类的基本结构
- 环境枚举 `Environment`
- 多环境文件加载逻辑
- 数据库 URL 构建属性
- NATS 配置（包含 validation_alias 支持云效）
- S3 兼容存储配置
- 临时目录配置

**需要参数化的配置项**：
```python
# 项目基本信息
PROJECT_NAME: str = "my_project"  # 占位符，初始化时替换

# 数据库配置
DB_USER: str  # 从环境变量读取
DB_PASSWORD: str
DB_HOST: str = "localhost"
DB_PORT: str = "5432"
DB_NAME: str = "my_project_dev"  # 占位符，初始化时替换

# JWT 配置
SECRET_KEY: str  # 初始化时自动生成随机值
ALGORITHM: str = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES: int = 1440

# NATS 配置
NATS_URL: str = "nats://localhost:4222"
NATS_STREAM: str = "my_project_tasks"  # 占位符，初始化时替换

# S3 配置
S3_ENDPOINT: str | None = None
S3_ACCESS_KEY_ID: str | None = None
S3_SECRET_ACCESS_KEY: str | None = None
S3_BUCKET: str | None = None  # 占位符，初始化时替换
S3_PREFIX: str | None = "my_project/dev"  # 占位符，初始化时替换
```

**简化内容**：
- 移除业务相关的配置（如 IMAGE_GENERATOR_* 配置）
- 保留通用的临时目录配置
- 保留 R2_PUBLIC_DOMAIN 配置（通用功能）


### 2. 数据库管理 (core/database.py)

**设计原则**：
- 使用 SQLModel + SQLAlchemy 异步引擎
- 提供连接池管理和自动重连
- 支持 FastAPI 依赖注入
- 提供启动时自动创建表的功能

**从 OneManage 完整复制**：
```python
# 核心组件（无需修改）
- async_engine: 异步数据库引擎
- AsyncSessionFactory: 会话工厂
- get_async_session(): FastAPI 依赖注入函数
- create_db_and_tables(): 启动时创建表（带重试）
- get_session_context(): 后台任务使用的上下文管理器
```

**连接池配置**：
```python
async_engine = create_async_engine(
    DATABASE_URL,
    echo=settings.DB_ECHO_LOG,
    future=True,
    pool_size=10,            # 基础连接池大小
    max_overflow=20,         # 允许的突发连接数
    pool_recycle=180,        # 连接回收时间（秒）
    pool_pre_ping=True,      # 连接前 ping 检查
    connect_args={"timeout": 15},  # 连接超时
)
```

**向后兼容**：
- 保留 `get_db` 别名指向 `get_async_session`
- 保留 `AsyncSessionLocal` 别名指向 `AsyncSessionFactory`


### 3. 消息队列配置 (core/broker.py)

**设计原则**：
- 使用 TaskIQ + NATS JetStream
- 支持双流架构（TASKS 流和 EVENTS 流）
- 自动发现并导入任务模块
- 禁用自动流创建（使用 setup_jetstream.py 预创建）

**从 OneManage 提取的内容**：
```python
# 核心组件
- CustomJetStreamBroker: 自定义 Broker 类（禁用自动流创建）
- tasks_broker: 命令任务 Broker（TASKS 流）
- events_broker: 事件监听 Broker（EVENTS 流）
- broker: 向后兼容别名
- _auto_import_tasks(): 自动发现任务模块
```

**Broker 配置**：
```python
# TASKS Broker
tasks_broker = CustomJetStreamBroker(
    servers=settings.nats_servers,
    stream_name="TASKS",
    subject="tasks.default",
    **nats_kwargs,  # user/password 如果配置了
)

# EVENTS Broker
events_broker = CustomJetStreamBroker(
    servers=settings.nats_servers,
    stream_name="EVENTS",
    subject="events.>",
    **nats_kwargs,
)
```

**启动钩子**：
- 为两个 Broker 注册 WORKER_STARTUP 和 WORKER_SHUTDOWN 事件
- 记录启动和关闭日志

**自动任务导入**：
- 扫描 `features/` 目录下的所有 `*tasks.py` 文件
- 自动导入，确保 `@broker.task` 装饰器正确注册


### 4. 对象存储客户端 (common/oss.py)

**设计原则**：
- 使用 aioboto3 实现 S3 兼容客户端
- 支持阿里云 OSS 和 Cloudflare R2
- 提供异步上传、下载、删除、预签名 URL 功能
- 支持公开域名和签名 URL 两种访问方式

**从 OneManage 提取的内容**：
```python
# 核心类和方法
- OSSClient: S3 兼容存储客户端
  - upload_file(): 异步上传文件
  - download_file(): 异步下载文件
  - delete_file(): 删除文件
  - generate_presigned_url(): 生成预签名 URL
  - get_public_url(): 获取公开访问 URL
  - list_objects(): 列举对象
```

**配置参数**：
```python
class OSSClient:
    def __init__(
        self,
        endpoint: str,
        access_key_id: str,
        secret_access_key: str,
        bucket: str,
        region: str = "auto",
        prefix: str | None = None,
        public_domain: str | None = None,
    ):
        # 从 settings 读取配置
```

**使用示例**：
```python
from common.oss import OSSClient
from core.config import settings

# 创建客户端实例
oss_client = OSSClient(
    endpoint=settings.S3_ENDPOINT,
    access_key_id=settings.S3_ACCESS_KEY_ID,
    secret_access_key=settings.S3_SECRET_ACCESS_KEY,
    bucket=settings.S3_BUCKET,
    prefix=settings.S3_PREFIX,
    public_domain=settings.R2_PUBLIC_DOMAIN,
)

# 上传文件
await oss_client.upload_file("local/path.png", "remote/key.png")

# 生成访问 URL
url = oss_client.get_public_url("remote/key.png")
```


### 5. 日志系统 (common/logging_config.py)

**设计原则**：
- 使用 Loguru 作为日志库
- 开发环境输出彩色格式化日志
- 生产环境输出 JSON 格式日志
- 支持日志级别配置

**从 OneManage 提取的内容**：
```python
# 日志配置函数
def setup_logging(env: str = "development"):
    """配置 Loguru 日志系统
    
    Args:
        env: 环境类型（development/production）
    """
    # 移除默认处理器
    logger.remove()
    
    if env == "development":
        # 开发环境：彩色格式化输出
        logger.add(
            sys.stderr,
            format="<green>{time:YYYY-MM-DD HH:mm:ss}</green> | <level>{level: <8}</level> | <cyan>{name}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> - <level>{message}</level>",
            level="DEBUG",
            colorize=True,
        )
    else:
        # 生产环境：JSON 格式输出
        logger.add(
            sys.stderr,
            format="{message}",
            level="INFO",
            serialize=True,  # JSON 格式
        )
```

**在 main.py 中调用**：
```python
from common.logging_config import setup_logging
from core.config import settings

# 应用启动时配置日志
setup_logging(env=settings.ENV.value)
```


### 6. 统一响应格式 (common/response_envelope.py + envelope_middleware.py)

**设计原则**：
- 所有 API 响应使用统一格式
- 成功响应包含 code、message、data
- 错误响应包含 code、message、error
- 使用中间件自动包装响应

**响应格式定义** (response_envelope.py):
```python
from pydantic import BaseModel
from typing import Any, Generic, TypeVar

T = TypeVar("T")

class ResponseEnvelope(BaseModel, Generic[T]):
    """统一响应格式"""
    code: int = 200
    message: str = "success"
    data: T | None = None
    error: str | None = None

# 成功响应示例
{
    "code": 200,
    "message": "success",
    "data": {"id": 1, "name": "example"}
}

# 错误响应示例
{
    "code": 400,
    "message": "Bad Request",
    "error": "Invalid parameter: name is required"
}
```

**中间件实现** (envelope_middleware.py):
```python
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware

class ResponseEnvelopeMiddleware(BaseHTTPMiddleware):
    """自动包装 API 响应的中间件"""
    
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # 如果响应已经是包装格式，直接返回
        # 否则包装成统一格式
        
        return response
```

**在 main.py 中注册**：
```python
from common.envelope_middleware import ResponseEnvelopeMiddleware

app.add_middleware(ResponseEnvelopeMiddleware)
```


### 7. 事件驱动架构 (events/)

**设计原则**：
- 使用 NATS JetStream 作为事件总线
- 事件发布和订阅解耦
- 事件 Schema 使用 Pydantic 验证
- 主题命名遵循 `events.{domain}.{action}.{status}` 格式

**事件主题定义** (events/subjects.py):
```python
"""NATS 事件主题定义"""

# 示例事件主题
EXAMPLE_CREATED = "events.example.created"
EXAMPLE_UPDATED = "events.example.updated"
EXAMPLE_DELETED = "events.example.deleted"
```

**事件 Schema 定义** (events/schemas.py):
```python
from pydantic import BaseModel, Field
from datetime import datetime

class BaseEvent(BaseModel):
    """事件基类"""
    event_id: str = Field(..., description="事件ID")
    timestamp: datetime = Field(default_factory=datetime.utcnow, description="事件时间戳")

class ExampleCreatedEvent(BaseEvent):
    """示例创建事件"""
    example_id: str
    name: str
```

**事件发布器** (events/publisher.py):
```python
from core.broker import broker
from loguru import logger

async def publish_event(subject: str, event_data: dict):
    """发布事件到 NATS
    
    Args:
        subject: 事件主题
        event_data: 事件数据（字典）
    """
    try:
        await broker.client.publish(subject, event_data)
        logger.info(f"[Event] 发布事件: {subject}")
    except Exception as e:
        logger.error(f"[Event] 发布事件失败: {subject}, error={e}")
        raise
```

**事件订阅示例** (features/example/tasks.py):
```python
from core.broker import events_broker
from events.subjects import EXAMPLE_CREATED

@events_broker.task
async def on_example_created(event_data: dict):
    """处理示例创建事件"""
    logger.info(f"[Event] 收到示例创建事件: {event_data}")
    # 处理事件逻辑
```


## 工具脚本设计

### 1. 项目初始化脚本 (scripts/init_project.py)

**功能**：
- 接受命令行参数（项目名称、数据库名称、存储桶等）
- 复制 `.env.example` 到 `.env.development`
- 替换配置文件中的占位符
- 生成随机 SECRET_KEY
- 更新 `pyproject.toml` 中的项目信息

**使用方式**：
```bash
pdm run python scripts/init_project.py \
  --project-name my_project \
  --db-name my_project_dev \
  --bucket my-bucket \
  --nats-stream my_project_tasks
```

**实现逻辑**：
```python
import argparse
import secrets
from pathlib import Path

def init_project(
    project_name: str,
    db_name: str,
    bucket: str | None = None,
    nats_stream: str | None = None,
):
    """初始化项目配置"""
    
    # 1. 复制 .env.example 到 .env.development
    # 2. 替换占位符
    # 3. 生成随机 SECRET_KEY
    # 4. 更新 pyproject.toml
    # 5. 输出下一步指引
```


### 2. 数据库连接测试 (scripts/test_db_connection.py)

**功能**：
- 测试数据库连接是否正常
- 执行简单查询获取数据库版本
- 输出详细的错误信息和配置检查建议

**使用方式**：
```bash
pdm run python scripts/test_db_connection.py
```

**实现逻辑**：
```python
import asyncio
from sqlalchemy import text
from core.database import async_engine
from core.config import settings
from loguru import logger

async def test_connection():
    """测试数据库连接"""
    try:
        logger.info("🔍 测试数据库连接...")
        logger.info(f"   数据库地址: {settings.DB_HOST}:{settings.DB_PORT}")
        logger.info(f"   数据库名称: {settings.DB_NAME}")
        logger.info(f"   用户名: {settings.DB_USER}")
        
        async with async_engine.connect() as conn:
            result = await conn.execute(text("SELECT version()"))
            version = result.scalar()
            
            logger.success("✅ 数据库连接成功！")
            logger.info(f"   数据库版本: {version}")
            
    except Exception as e:
        logger.error("❌ 数据库连接失败！")
        logger.error(f"   错误信息: {str(e)}")
        logger.info("\n📋 配置检查建议:")
        logger.info("   1. 检查数据库服务是否启动")
        logger.info("   2. 检查 DB_HOST 和 DB_PORT 是否正确")
        logger.info("   3. 检查 DB_USER 和 DB_PASSWORD 是否正确")
        logger.info("   4. 检查数据库 DB_NAME 是否存在")
        raise

if __name__ == "__main__":
    asyncio.run(test_connection())
```


### 3. OSS 连接测试 (scripts/test_oss_connection.py)

**功能**：
- 测试 OSS 连接是否正常
- 列举存储桶内容
- 上传测试文件并删除
- 输出详细的错误信息和配置检查建议

**使用方式**：
```bash
pdm run python scripts/test_oss_connection.py
```

**实现逻辑**：
```python
import asyncio
from pathlib import Path
from common.oss import OSSClient
from core.config import settings
from loguru import logger

async def test_oss():
    """测试 OSS 连接"""
    try:
        logger.info("🔍 测试 OSS 连接...")
        logger.info(f"   Endpoint: {settings.S3_ENDPOINT}")
        logger.info(f"   Bucket: {settings.S3_BUCKET}")
        logger.info(f"   Prefix: {settings.S3_PREFIX}")
        
        # 创建客户端
        client = OSSClient(
            endpoint=settings.S3_ENDPOINT,
            access_key_id=settings.S3_ACCESS_KEY_ID,
            secret_access_key=settings.S3_SECRET_ACCESS_KEY,
            bucket=settings.S3_BUCKET,
            prefix=settings.S3_PREFIX,
        )
        
        # 列举对象
        objects = await client.list_objects(max_keys=5)
        logger.info(f"   存储桶对象数: {len(objects)}")
        
        # 上传测试文件
        test_key = "test/connection_test.txt"
        test_content = b"OSS connection test"
        await client.upload_file(test_content, test_key)
        logger.info(f"   测试文件上传成功: {test_key}")
        
        # 删除测试文件
        await client.delete_file(test_key)
        logger.info(f"   测试文件删除成功")
        
        logger.success("✅ OSS 连接测试通过！")
        
    except Exception as e:
        logger.error("❌ OSS 连接失败！")
        logger.error(f"   错误信息: {str(e)}")
        logger.info("\n📋 配置检查建议:")
        logger.info("   1. 检查 S3_ENDPOINT 是否正确")
        logger.info("   2. 检查 S3_ACCESS_KEY_ID 和 S3_SECRET_ACCESS_KEY 是否正确")
        logger.info("   3. 检查 S3_BUCKET 是否存在")
        logger.info("   4. 检查网络连接是否正常")
        raise

if __name__ == "__main__":
    asyncio.run(test_oss())
```


### 4. NATS 连接测试 (scripts/test_nats_connection.py)

**功能**：
- 测试 NATS 服务器连接
- 发布测试消息
- 验证 JetStream 流是否存在

**使用方式**：
```bash
pdm run python scripts/test_nats_connection.py
```

**实现逻辑**：
```python
import asyncio
from nats.aio.client import Client as NATS
from core.config import settings
from loguru import logger

async def test_nats():
    """测试 NATS 连接"""
    nc = NATS()
    
    try:
        logger.info("🔍 测试 NATS 连接...")
        logger.info(f"   NATS URL: {settings.NATS_URL}")
        
        # 连接 NATS
        await nc.connect(servers=[settings.NATS_URL])
        logger.info("   ✅ NATS 连接成功")
        
        # 获取 JetStream 上下文
        js = nc.jetstream()
        
        # 检查流是否存在
        try:
            stream_info = await js.stream_info("TASKS")
            logger.info(f"   ✅ TASKS 流存在，消息数: {stream_info.state.messages}")
        except Exception:
            logger.warning("   ⚠️  TASKS 流不存在，请运行: pdm run python scripts/setup_jetstream.py")
        
        try:
            stream_info = await js.stream_info("EVENTS")
            logger.info(f"   ✅ EVENTS 流存在，消息数: {stream_info.state.messages}")
        except Exception:
            logger.warning("   ⚠️  EVENTS 流不存在，请运行: pdm run python scripts/setup_jetstream.py")
        
        logger.success("✅ NATS 连接测试通过！")
        
    except Exception as e:
        logger.error("❌ NATS 连接失败！")
        logger.error(f"   错误信息: {str(e)}")
        logger.info("\n📋 配置检查建议:")
        logger.info("   1. 检查 NATS 服务是否启动")
        logger.info("   2. 检查 NATS_URL 是否正确")
        logger.info("   3. 如果使用认证，检查 NATS_USER 和 NATS_PASSWORD")
        raise
    finally:
        await nc.close()

if __name__ == "__main__":
    asyncio.run(test_nats())
```


### 5. 一键健康检查 (scripts/health_check.py)

**功能**：
- 依次检查数据库、NATS、OSS 的连接状态
- 输出汇总报告
- 返回非零退出码如果有任何检查失败

**使用方式**：
```bash
pdm run python scripts/health_check.py
```

**实现逻辑**：
```python
import asyncio
import sys
from loguru import logger

async def check_database():
    """检查数据库连接"""
    try:
        from scripts.test_db_connection import test_connection
        await test_connection()
        return True
    except Exception:
        return False

async def check_nats():
    """检查 NATS 连接"""
    try:
        from scripts.test_nats_connection import test_nats
        await test_nats()
        return True
    except Exception:
        return False

async def check_oss():
    """检查 OSS 连接"""
    try:
        from scripts.test_oss_connection import test_oss
        await test_oss()
        return True
    except Exception:
        return False

async def main():
    """执行所有健康检查"""
    logger.info("=" * 60)
    logger.info("🏥 开始健康检查...")
    logger.info("=" * 60)
    
    results = {
        "数据库": await check_database(),
        "NATS": await check_nats(),
        "OSS": await check_oss(),
    }
    
    logger.info("\n" + "=" * 60)
    logger.info("📊 健康检查汇总:")
    logger.info("=" * 60)
    
    all_passed = True
    for service, passed in results.items():
        status = "✅ 通过" if passed else "❌ 失败"
        logger.info(f"   {service}: {status}")
        if not passed:
            all_passed = False
    
    logger.info("=" * 60)
    
    if all_passed:
        logger.success("🎉 所有服务健康检查通过！")
        sys.exit(0)
    else:
        logger.error("⚠️  部分服务健康检查失败，请检查配置")
        sys.exit(1)

if __name__ == "__main__":
    asyncio.run(main())
```


### 6. NATS JetStream 初始化 (scripts/setup_jetstream.py)

**功能**：
- 创建 TASKS 和 EVENTS 两个 JetStream 流
- 配置流的保留策略和消息限制
- 幂等操作（流已存在则跳过）

**从 OneManage 完整复制**：
```python
import asyncio
from nats.aio.client import Client as NATS
from nats.js.api import StreamConfig, RetentionPolicy
from core.config import settings
from loguru import logger

async def setup_jetstream():
    """初始化 NATS JetStream 流"""
    nc = NATS()
    
    try:
        await nc.connect(servers=[settings.NATS_URL])
        js = nc.jetstream()
        
        # 创建 TASKS 流
        tasks_config = StreamConfig(
            name="TASKS",
            subjects=["tasks.>"],
            retention=RetentionPolicy.WORK_QUEUE,
            max_msgs=10000,
            max_age=86400,  # 24小时
        )
        
        try:
            await js.add_stream(config=tasks_config)
            logger.info("✅ TASKS 流创建成功")
        except Exception as e:
            if "already exists" in str(e):
                logger.info("ℹ️  TASKS 流已存在")
            else:
                raise
        
        # 创建 EVENTS 流
        events_config = StreamConfig(
            name="EVENTS",
            subjects=["events.>"],
            retention=RetentionPolicy.LIMITS,
            max_msgs=100000,
            max_age=604800,  # 7天
        )
        
        try:
            await js.add_stream(config=events_config)
            logger.info("✅ EVENTS 流创建成功")
        except Exception as e:
            if "already exists" in str(e):
                logger.info("ℹ️  EVENTS 流已存在")
            else:
                raise
        
        logger.success("🎉 JetStream 初始化完成！")
        
    finally:
        await nc.close()

if __name__ == "__main__":
    asyncio.run(setup_jetstream())
```


## 示例业务模块设计

### 目录结构

```
features/example/
├── __init__.py
├── models.py       # SQLModel 数据模型
├── schemas.py      # Pydantic Schema
├── routers.py      # FastAPI 路由
├── services.py     # 业务逻辑
└── tasks.py        # TaskIQ 异步任务
```

### 1. 数据模型 (models.py)

```python
from sqlmodel import SQLModel, Field
from datetime import datetime
from typing import Optional

class Example(SQLModel, table=True):
    """示例数据模型"""
    __tablename__ = "examples"
    
    id: Optional[int] = Field(default=None, primary_key=True)
    name: str = Field(..., max_length=100, description="名称")
    description: Optional[str] = Field(None, description="描述")
    created_at: datetime = Field(default_factory=datetime.utcnow, description="创建时间")
    updated_at: datetime = Field(default_factory=datetime.utcnow, description="更新时间")
```

### 2. Schema 定义 (schemas.py)

```python
from pydantic import BaseModel
from datetime import datetime

class ExampleCreate(BaseModel):
    """创建示例请求"""
    name: str
    description: str | None = None

class ExampleUpdate(BaseModel):
    """更新示例请求"""
    name: str | None = None
    description: str | None = None

class ExampleResponse(BaseModel):
    """示例响应"""
    id: int
    name: str
    description: str | None
    created_at: datetime
    updated_at: datetime
    
    class Config:
        from_attributes = True
```


### 3. 路由定义 (routers.py)

```python
from fastapi import APIRouter, Depends
from sqlmodel.ext.asyncio.session import AsyncSession
from core.database import get_db
from .schemas import ExampleCreate, ExampleUpdate, ExampleResponse
from .services import ExampleService

router = APIRouter(prefix="/examples", tags=["Examples"])

@router.post("/", response_model=ExampleResponse)
async def create_example(
    data: ExampleCreate,
    db: AsyncSession = Depends(get_db),
):
    """创建示例"""
    service = ExampleService(db)
    return await service.create(data)

@router.get("/{example_id}", response_model=ExampleResponse)
async def get_example(
    example_id: int,
    db: AsyncSession = Depends(get_db),
):
    """获取示例"""
    service = ExampleService(db)
    return await service.get_by_id(example_id)

@router.put("/{example_id}", response_model=ExampleResponse)
async def update_example(
    example_id: int,
    data: ExampleUpdate,
    db: AsyncSession = Depends(get_db),
):
    """更新示例"""
    service = ExampleService(db)
    return await service.update(example_id, data)

@router.delete("/{example_id}")
async def delete_example(
    example_id: int,
    db: AsyncSession = Depends(get_db),
):
    """删除示例"""
    service = ExampleService(db)
    await service.delete(example_id)
    return {"message": "删除成功"}
```


### 4. 业务逻辑 (services.py)

```python
from sqlmodel import select
from sqlmodel.ext.asyncio.session import AsyncSession
from fastapi import HTTPException
from .models import Example
from .schemas import ExampleCreate, ExampleUpdate
from events.publisher import publish_event
from events.subjects import EXAMPLE_CREATED
from loguru import logger

class ExampleService:
    """示例业务逻辑"""
    
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def create(self, data: ExampleCreate) -> Example:
        """创建示例"""
        example = Example(**data.model_dump())
        self.db.add(example)
        await self.db.commit()
        await self.db.refresh(example)
        
        # 发布创建事件
        await publish_event(EXAMPLE_CREATED, {
            "example_id": example.id,
            "name": example.name,
        })
        
        logger.info(f"[Example] 创建成功: id={example.id}")
        return example
    
    async def get_by_id(self, example_id: int) -> Example:
        """根据 ID 获取示例"""
        result = await self.db.execute(
            select(Example).where(Example.id == example_id)
        )
        example = result.scalar_one_or_none()
        
        if not example:
            raise HTTPException(status_code=404, detail="示例不存在")
        
        return example
    
    async def update(self, example_id: int, data: ExampleUpdate) -> Example:
        """更新示例"""
        example = await self.get_by_id(example_id)
        
        update_data = data.model_dump(exclude_unset=True)
        for key, value in update_data.items():
            setattr(example, key, value)
        
        await self.db.commit()
        await self.db.refresh(example)
        
        logger.info(f"[Example] 更新成功: id={example_id}")
        return example
    
    async def delete(self, example_id: int):
        """删除示例"""
        example = await self.get_by_id(example_id)
        await self.db.delete(example)
        await self.db.commit()
        
        logger.info(f"[Example] 删除成功: id={example_id}")
```


### 5. 异步任务 (tasks.py)

```python
from core.broker import broker, events_broker
from events.subjects import EXAMPLE_CREATED
from loguru import logger

@broker.task
async def task_process_example(example_id: int, name: str):
    """处理示例的异步任务（命令任务）
    
    这是一个命令任务，使用 tasks_broker（TASKS 流）
    """
    logger.info(f"[Task] 开始处理示例: id={example_id}, name={name}")
    
    # 模拟耗时操作
    import asyncio
    await asyncio.sleep(2)
    
    logger.info(f"[Task] 处理完成: id={example_id}")
    return {"status": "success", "example_id": example_id}

@events_broker.task
async def on_example_created(event_data: dict):
    """监听示例创建事件（事件处理器）
    
    这是一个事件处理器，使用 events_broker（EVENTS 流）
    """
    example_id = event_data.get("example_id")
    name = event_data.get("name")
    
    logger.info(f"[Event] 收到示例创建事件: id={example_id}, name={name}")
    
    # 触发后续处理任务
    await task_process_example.kiq(example_id=example_id, name=name)
```


## 应用入口设计 (main.py)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from core.config import settings
from core.database import create_db_and_tables
from common.logging_config import setup_logging
from common.envelope_middleware import ResponseEnvelopeMiddleware

# 配置日志
setup_logging(env=settings.ENV.value)

@asynccontextmanager
async def lifespan(app: FastAPI):
    """应用生命周期管理"""
    # 启动时
    await create_db_and_tables()
    yield
    # 关闭时
    pass

# 创建 FastAPI 应用
app = FastAPI(
    title=settings.PROJECT_NAME,
    version="0.1.0",
    lifespan=lifespan,
)

# 添加 CORS 中间件
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"] if settings.ENV == "development" else [],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 添加响应包装中间件
app.add_middleware(ResponseEnvelopeMiddleware)

# 注册路由
from features.example.routers import router as example_router
app.include_router(example_router)

@app.get("/")
async def root():
    """根路径"""
    return {"message": "Welcome to FastAPI Starter Template"}

@app.get("/health")
async def health():
    """健康检查端点"""
    return {"status": "healthy", "env": settings.ENV.value}
```


## 配置文件设计

### 1. .env.example

```bash
# 项目配置
PROJECT_NAME=my_project
ENV=development

# 数据库配置
DB_USER=postgres
DB_PASSWORD=your_password
DB_HOST=localhost
DB_PORT=5432
DB_NAME=my_project_dev

# JWT 配置
SECRET_KEY=your_secret_key_here
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=1440

# NATS 配置
NATS_URL=nats://localhost:4222
NATS_STREAM=my_project_tasks
NATS_USER=
NATS_PASSWORD=

# S3 兼容存储配置
S3_ENDPOINT=https://oss-cn-hangzhou.aliyuncs.com
S3_ACCESS_KEY_ID=your_access_key
S3_SECRET_ACCESS_KEY=your_secret_key
S3_BUCKET=my-bucket
S3_PREFIX=my_project/dev
S3_REGION=cn-hangzhou

# Cloudflare R2 公开域名（可选）
R2_PUBLIC_DOMAIN=

# 应用配置
DEBUG=true
DB_ECHO_LOG=false
TEMP_DIR=/tmp
```


### 2. pyproject.toml

```toml
[project]
name = "fastapi-starter-template"
version = "0.1.0"
description = "FastAPI project template with database, message queue, and object storage"
authors = [{name = "Your Name", email = "your@email.com"}]
requires-python = ">=3.11"
dependencies = [
    # Web 框架
    "fastapi>=0.104.0",
    "uvicorn[standard]>=0.24.0",
    
    # 数据库
    "sqlmodel>=0.0.14",
    "asyncpg>=0.29.0",
    "alembic>=1.12.0",
    
    # 消息队列
    "taskiq>=0.11.0",
    "taskiq-nats>=0.4.0",
    "nats-py>=2.6.0",
    
    # 对象存储
    "aioboto3>=12.0.0",
    "boto3>=1.28.0",
    
    # 工具库
    "loguru>=0.7.0",
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",
    "python-dotenv>=1.0.0",
]

[project.optional-dependencies]
dev = [
    "black>=23.0.0",
    "ruff>=0.1.0",
    "pytest>=7.4.0",
    "pytest-asyncio>=0.21.0",
]

[tool.pdm]
distribution = false

[tool.black]
line-length = 120
target-version = ["py311"]

[tool.ruff]
line-length = 120
target-version = "py311"
select = ["E", "F", "I"]
ignore = ["E501"]

[build-system]
requires = ["pdm-backend"]
build-backend = "pdm.backend"
```


### 3. alembic.ini

```ini
[alembic]
script_location = migrations
prepend_sys_path = .
version_path_separator = os

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S
```


### 4. .gitignore

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
*.egg-info/
.installed.cfg
*.egg

# 虚拟环境
venv/
ENV/
env/
.venv

# PDM
.pdm.toml
.pdm-python
__pypackages__/

# 环境变量
.env
.env.*
!.env.example

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# 日志
*.log

# 数据库
*.db
*.sqlite

# 临时文件
/tmp/
*.tmp

# 操作系统
.DS_Store
Thumbs.db
```


## 数据库迁移设计

### migrations/env.py

从 OneManage 提取并简化，核心功能：
- 从 core.config 读取数据库 URL
- 支持异步迁移
- 自动发现所有 SQLModel 模型

```python
import asyncio
from logging.config import fileConfig
from sqlalchemy import pool
from sqlalchemy.engine import Connection
from sqlalchemy.ext.asyncio import async_engine_from_config
from alembic import context
from sqlmodel import SQLModel

# 导入配置
from core.config import settings

# Alembic Config 对象
config = context.config

# 设置数据库 URL
config.set_main_option("sqlalchemy.url", str(settings.database_url))

# 配置日志
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# 导入所有模型（确保 SQLModel.metadata 包含所有表）
from features.example.models import Example  # noqa

# 目标元数据
target_metadata = SQLModel.metadata

def run_migrations_offline() -> None:
    """离线模式运行迁移"""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()

def do_run_migrations(connection: Connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)

    with context.begin_transaction():
        context.run_migrations()

async def run_async_migrations() -> None:
    """异步模式运行迁移"""
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)

    await connectable.dispose()

def run_migrations_online() -> None:
    """在线模式运行迁移"""
    asyncio.run(run_async_migrations())

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```


## 使用流程

### 1. 克隆模板

```bash
cd ~/coding
git clone <template-repo-url> my-new-project
cd my-new-project
```

### 2. 初始化项目

```bash
# 安装依赖
pdm install

# 运行初始化脚本
pdm run python scripts/init_project.py \
  --project-name my_project \
  --db-name my_project_dev \
  --bucket my-bucket \
  --nats-stream my_project_tasks
```

初始化脚本会：
- 复制 `.env.example` 到 `.env.development`
- 替换所有占位符
- 生成随机 SECRET_KEY
- 更新 `pyproject.toml`

### 3. 配置环境变量

编辑 `.env.development`，填写实际的配置值：
```bash
DB_PASSWORD=your_actual_password
S3_ACCESS_KEY_ID=your_actual_key
S3_SECRET_ACCESS_KEY=your_actual_secret
```

### 4. 验证配置

```bash
# 一键健康检查
pdm run python scripts/health_check.py

# 或单独测试
pdm run python scripts/test_db_connection.py
pdm run python scripts/test_nats_connection.py
pdm run python scripts/test_oss_connection.py
```

### 5. 初始化 NATS JetStream

```bash
pdm run python scripts/setup_jetstream.py
```

### 6. 运行数据库迁移

```bash
# 生成初始迁移
pdm run alembic revision --autogenerate -m "Initial migration"

# 应用迁移
pdm run alembic upgrade head
```

### 7. 启动应用

```bash
# 启动 API 服务器
pdm run uvicorn main:app --reload

# 启动 TaskIQ Worker（另一个终端）
pdm run taskiq worker core.broker:broker --workers 1
```

### 8. 测试 API

```bash
# 访问文档
open http://localhost:8000/docs

# 测试健康检查
curl http://localhost:8000/health

# 创建示例
curl -X POST http://localhost:8000/examples/ \
  -H "Content-Type: application/json" \
  -d '{"name": "test", "description": "test example"}'
```


## 从 OneManage 提取文件清单

### 完整复制（无需修改）

| 源文件 | 目标文件 | 说明 |
|--------|---------|------|
| `OneManage/core/database.py` | `core/database.py` | 数据库连接管理 |
| `OneManage/common/json_utils.py` | `common/json_utils.py` | JSON 序列化工具 |
| `OneManage/common/oss.py` | `common/oss.py` | OSS 客户端 |
| `OneManage/migrations/script.py.mako` | `migrations/script.py.mako` | 迁移脚本模板 |

### 需要简化的文件

| 源文件 | 目标文件 | 简化内容 |
|--------|---------|---------|
| `OneManage/core/config.py` | `core/config.py` | 移除业务相关配置（IMAGE_GENERATOR_*等） |
| `OneManage/core/broker.py` | `core/broker.py` | 保留核心功能，移除业务任务导入 |
| `OneManage/migrations/env.py` | `migrations/env.py` | 移除业务模型导入，只保留示例模型 |

### 需要新建的文件

| 文件路径 | 说明 |
|---------|------|
| `common/logging_config.py` | 日志配置（从 OneManage 提取逻辑） |
| `common/response_envelope.py` | 统一响应格式 |
| `common/envelope_middleware.py` | 响应包装中间件 |
| `events/__init__.py` | 事件模块初始化 |
| `events/publisher.py` | 事件发布器 |
| `events/schemas.py` | 事件 Schema |
| `events/subjects.py` | 事件主题定义 |
| `scripts/init_project.py` | 项目初始化脚本 |
| `scripts/test_db_connection.py` | 数据库测试脚本 |
| `scripts/test_oss_connection.py` | OSS 测试脚本 |
| `scripts/test_nats_connection.py` | NATS 测试脚本 |
| `scripts/health_check.py` | 健康检查脚本 |
| `scripts/setup_jetstream.py` | JetStream 初始化（从 OneManage 复制） |
| `features/example/models.py` | 示例数据模型 |
| `features/example/schemas.py` | 示例 Schema |
| `features/example/routers.py` | 示例路由 |
| `features/example/services.py` | 示例业务逻辑 |
| `features/example/tasks.py` | 示例异步任务 |
| `main.py` | FastAPI 应用入口 |
| `.env.example` | 配置模板 |
| `pyproject.toml` | 依赖管理 |
| `alembic.ini` | Alembic 配置 |
| `.gitignore` | Git 忽略规则 |


## 技术决策

### 1. 为什么不包含认证系统？

- 认证系统通常是业务相关的，不同项目有不同需求
- 有些项目使用 JWT，有些使用 OAuth2，有些使用 API Key
- 保持模板最小化，让开发者根据需求添加认证

### 2. 为什么使用双 Broker 架构？

- 命令任务（TASKS）和事件处理（EVENTS）有不同的语义
- TASKS 流使用 WORK_QUEUE 模式（1:1 消费）
- EVENTS 流使用 LIMITS 模式（1:N 广播）
- 分离可以更好地管理消息保留策略

### 3. 为什么不包含 Docker 配置？

- Docker 配置通常需要根据部署环境定制
- 本地开发可以直接使用系统服务（PostgreSQL、NATS）
- 生产环境通常使用云服务（RDS、云消息队列）
- 保持模板简洁，避免过度配置

### 4. 为什么使用 PDM 而不是 Poetry？

- 与 OneManage 保持一致
- PDM 更符合 PEP 标准
- 性能更好，依赖解析更快

### 5. 为什么包含 OSS 客户端？

- 对象存储是现代应用的标准组件
- S3 兼容 API 是事实标准
- 支持阿里云 OSS 和 Cloudflare R2，覆盖常见场景

### 6. 为什么使用 Loguru 而不是标准 logging？

- Loguru API 更简洁易用
- 自动支持彩色输出和 JSON 格式
- 与 OneManage 保持一致


## 扩展指南

### 添加新的业务模块

1. 在 `features/` 下创建新目录
2. 按照示例模块的结构创建文件
3. 在 `main.py` 中注册路由

```bash
features/
└── my_module/
    ├── __init__.py
    ├── models.py
    ├── schemas.py
    ├── routers.py
    ├── services.py
    └── tasks.py
```

### 添加新的事件

1. 在 `events/subjects.py` 中定义主题
2. 在 `events/schemas.py` 中定义 Schema
3. 在业务逻辑中发布事件
4. 在任务模块中订阅事件

### 添加认证系统

1. 创建 `features/auth/` 模块
2. 实现 JWT 或 OAuth2 认证
3. 创建依赖注入函数（如 `get_current_user`）
4. 在需要认证的路由中使用依赖

### 添加缓存层

1. 安装 Redis 客户端：`pdm add redis[hiredis]`
2. 在 `core/` 中创建 `cache.py`
3. 实现缓存装饰器或工具函数
4. 在服务层使用缓存

### 添加定时任务

1. 安装 APScheduler：`pdm add apscheduler`
2. 在 `core/` 中创建 `scheduler.py`
3. 在 `main.py` 的 lifespan 中启动调度器
4. 定义定时任务函数


## 测试策略

### 单元测试

虽然模板不包含测试代码，但建议的测试结构：

```
tests/
├── conftest.py              # pytest 配置和 fixtures
├── test_config.py           # 配置测试
├── test_database.py         # 数据库测试
├── test_oss.py              # OSS 客户端测试
└── features/
    └── example/
        ├── test_models.py   # 模型测试
        ├── test_services.py # 服务测试
        └── test_routers.py  # 路由测试
```

### 集成测试

建议使用 pytest-asyncio 进行异步测试：

```python
import pytest
from httpx import AsyncClient
from main import app

@pytest.mark.asyncio
async def test_create_example():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post(
            "/examples/",
            json={"name": "test", "description": "test"}
        )
        assert response.status_code == 200
```

### 测试数据库

建议使用独立的测试数据库：

```python
# conftest.py
@pytest.fixture
async def test_db():
    # 创建测试数据库
    # 运行迁移
    # 返回会话
    # 清理
```


## 部署建议

### 开发环境

```bash
# 启动 API
pdm run uvicorn main:app --reload --host 0.0.0.0 --port 8000

# 启动 Worker
pdm run taskiq worker core.broker:broker --workers 1
```

### 生产环境

```bash
# 设置环境变量
export ENV=production

# 启动 API（使用 Gunicorn + Uvicorn）
pdm run gunicorn main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000

# 启动 Worker（使用进程管理器如 Supervisor）
pdm run taskiq worker core.broker:broker --workers 4
```

### 云效部署

1. 配置环境变量注入
2. 使用 RDS 作为数据库
3. 使用云消息队列（如阿里云 NATS）
4. 使用 OSS 作为对象存储
5. 配置健康检查端点：`/health`

### 容器化（可选）

虽然模板不包含 Dockerfile，但建议的 Dockerfile 结构：

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装 PDM
RUN pip install pdm

# 复制依赖文件
COPY pyproject.toml pdm.lock ./

# 安装依赖
RUN pdm install --prod --no-lock --no-editable

# 复制代码
COPY . .

# 暴露端口
EXPOSE 8000

# 启动命令
CMD ["pdm", "run", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```


## 常见问题

### Q1: 如何切换数据库？

A: 修改 `.env.development` 中的数据库配置，然后运行健康检查：
```bash
pdm run python scripts/test_db_connection.py
```

### Q2: 如何添加新的环境？

A: 创建新的环境文件（如 `.env.staging`），然后设置 ENV 环境变量：
```bash
export ENV=staging
pdm run uvicorn main:app
```

### Q3: 如何禁用 OSS 功能？

A: 在 `.env.development` 中不配置 S3_* 相关变量，OSS 客户端会返回 None。

### Q4: 如何查看 NATS 消息？

A: 使用 NATS CLI 工具：
```bash
nats stream ls
nats stream info TASKS
nats stream info EVENTS
```

### Q5: 如何重置数据库？

A: 使用 Alembic 降级到初始状态：
```bash
pdm run alembic downgrade base
pdm run alembic upgrade head
```

### Q6: 如何调试异步任务？

A: 在 Worker 启动时添加 `--log-level DEBUG`：
```bash
pdm run taskiq worker core.broker:broker --workers 1 --log-level DEBUG
```

### Q7: 如何更新依赖？

A: 使用 PDM 更新：
```bash
pdm update
pdm lock
```

### Q8: 如何生成 SECRET_KEY？

A: 使用 Python secrets 模块：
```python
import secrets
print(secrets.token_urlsafe(32))
```


## 总结

本设计文档详细描述了如何从 OneManage 项目中提取通用基础设施，创建一个独立的 FastAPI 项目模板。

### 核心特性

1. ✅ **配置管理**：Pydantic Settings + 多环境支持
2. ✅ **数据库**：SQLModel + Alembic + 异步连接池
3. ✅ **消息队列**：TaskIQ + NATS JetStream + 双流架构
4. ✅ **对象存储**：S3 兼容客户端（阿里云 OSS + Cloudflare R2）
5. ✅ **日志系统**：Loguru + 环境感知格式
6. ✅ **统一响应**：响应包装中间件
7. ✅ **事件驱动**：事件发布订阅基础设施
8. ✅ **验证脚本**：数据库、NATS、OSS 连接测试
9. ✅ **初始化脚本**：一键配置新项目
10. ✅ **示例模块**：完整的 CRUD + 异步任务示例

### 设计原则

- **最小化**：只包含通用基础设施
- **参数化**：所有配置可通过环境变量或脚本替换
- **可验证**：提供完整的验证脚本
- **可扩展**：清晰的模块结构，易于添加新功能
- **向后兼容**：与 OneManage 保持一致的技术栈

### 下一步

参考 `tasks.md` 文档，按照任务列表逐步实现模板项目。
