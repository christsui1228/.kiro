# FastAPI Starter Template 实施任务

## 任务优先级说明

- **P0 - 核心基础设施**：config.py 和 database.py（最高优先级）
- **P1 - 基础组件**：broker、OSS、日志等
- **P2 - 工具脚本**：验证和初始化脚本
- **P3 - 示例模块**：示例代码和文档

---

## P0 - 核心基础设施（最高优先级）

- [-] 1. 创建项目目录结构

**需求**: 1.1, 2.1, 3.1, 4.1, 5.1

**描述**: 在 `~/coding/fastapi-starter-template/` 创建完整的项目目录结构

**文件操作**:
- 创建根目录 `~/coding/fastapi-starter-template/`
- 创建 `core/` 目录及 `__init__.py`
- 创建 `common/` 目录及 `__init__.py`
- 创建 `events/` 目录及 `__init__.py`
- 创建 `scripts/` 目录及 `__init__.py`
- 创建 `migrations/` 目录
- 创建 `features/` 目录

**验收标准**:
- 所有目录创建成功
- 每个 Python 包目录包含 `__init__.py`

---

- [ ] 2. 创建核心配置管理 (core/config.py)

**需求**: 1.1, 1.2, 1.3, 1.4, 1.5

**描述**: 从 OneManage 提取并简化配置管理模块

**文件操作**:
- 复制 `OneManage/core/config.py` 到 `core/config.py`
- 移除业务相关配置（IMAGE_GENERATOR_*、FONT_DIR 等）
- 保留核心配置（数据库、NATS、S3、JWT）
- 添加 PROJECT_NAME 配置项
- 将硬编码值替换为占位符

**关键配置项**:
```python
# 项目基本信息
PROJECT_NAME: str = "fastapi_starter_template"

# 数据库配置
DB_USER: str
DB_PASSWORD: str
DB_HOST: str = "localhost"
DB_PORT: str = "5432"
DB_NAME: str = "starter_template_dev"

# JWT 配置
SECRET_KEY: str
ALGORITHM: str = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES: int = 1440

# NATS 配置
NATS_URL: str = Field(default="nats://localhost:4222", validation_alias="NATS_SERVER_URL")
NATS_STREAM: str = "starter_template_tasks"
NATS_USER: str | None = None
NATS_PASSWORD: str | None = None

# S3 配置
S3_ENDPOINT: str | None = None
S3_ACCESS_KEY_ID: str | None = None
S3_SECRET_ACCESS_KEY: str | None = None
S3_BUCKET: str | None = None
S3_PREFIX: str | None = "starter_template/dev"
S3_REGION: str = "auto"
R2_PUBLIC_DOMAIN: str | None = None

# 临时目录
TEMP_DIR: str = "/tmp"
```

**验收标准**:
- 配置类使用 Pydantic Settings
- 支持多环境文件加载（.env.development, .env.production）
- 包含 database_url 和 database_url_sync 属性
- 包含 nats_servers 属性
- 所有必需字段有清晰的类型注解

---

- [ ] 3. 创建数据库连接管理 (core/database.py)

**需求**: 2.1, 2.2, 2.3, 2.4

**描述**: 从 OneManage 完整复制数据库连接管理模块

**文件操作**:
- 完整复制 `OneManage/core/database.py` 到 `core/database.py`
- 无需修改，保持原样

**核心组件**:
```python
# 异步引擎配置
async_engine = create_async_engine(
    DATABASE_URL,
    echo=settings.DB_ECHO_LOG,
    future=True,
    pool_size=10,
    max_overflow=20,
    pool_recycle=180,
    pool_pre_ping=True,
    connect_args={"timeout": 15},
)

# 会话工厂
AsyncSessionFactory = sessionmaker(
    bind=async_engine,
    class_=AsyncSession,
    expire_on_commit=False,
    autoflush=False,
    autocommit=False,
)

# FastAPI 依赖注入
async def get_async_session() -> AsyncIterator[AsyncSession]:
    async with AsyncSessionFactory() as session:
        yield session

# 启动时创建表（带重试）
async def create_db_and_tables(max_attempts: int = 5, delay: int = 5):
    # 实现重试逻辑
```

**验收标准**:
- 提供 async_engine 异步引擎
- 提供 AsyncSessionFactory 会话工厂
- 提供 get_async_session() 依赖注入函数
- 提供 create_db_and_tables() 启动函数
- 提供 get_session_context() 上下文管理器
- 包含向后兼容别名（get_db, AsyncSessionLocal）

---

## P1 - 基础组件

- [ ] 4. 创建消息队列配置 (core/broker.py)

**需求**: 3.1, 3.2, 3.3, 3.4

**描述**: 从 OneManage 提取并简化 Broker 配置

**文件操作**:
- 复制 `OneManage/core/broker.py` 到 `core/broker.py`
- 保留 CustomJetStreamBroker 类
- 保留双 Broker 架构（tasks_broker, events_broker）
- 移除业务任务自动导入逻辑（_auto_import_tasks）

**核心组件**:
```python
# 自定义 Broker（禁用自动流创建）
class CustomJetStreamBroker(PushBasedJetStreamBroker):
    async def startup(self) -> None:
        # 跳过 add_stream() 调用
        pass

# TASKS Broker
tasks_broker = CustomJetStreamBroker(
    servers=settings.nats_servers,
    stream_name="TASKS",
    subject="tasks.default",
    **nats_kwargs,
)

# EVENTS Broker
events_broker = CustomJetStreamBroker(
    servers=settings.nats_servers,
    stream_name="EVENTS",
    subject="events.>",
    **nats_kwargs,
)

# 向后兼容
broker = tasks_broker
```

**验收标准**:
- 提供 tasks_broker 和 events_broker
- 提供 broker 向后兼容别名
- 注册启动和关闭钩子
- 支持 API 和 Worker 两种模式

---

- [ ] 5. 创建对象存储客户端 (common/oss.py)

**需求**: 4.1, 4.2, 4.3, 4.4, 4.5

**描述**: 从 OneManage 完整复制 OSS 客户端

**文件操作**:
- 完整复制 `OneManage/common/oss.py` 到 `common/oss.py`
- 无需修改

**核心功能**:
- upload_file(): 异步上传文件
- download_file(): 异步下载文件
- delete_file(): 删除文件
- generate_presigned_url(): 生成预签名 URL
- get_public_url(): 获取公开访问 URL
- list_objects(): 列举对象

**验收标准**:
- 支持阿里云 OSS 和 Cloudflare R2
- 所有方法都是异步的
- 支持公开域名和签名 URL

---

- [ ] 6. 创建日志配置 (common/logging_config.py)

**需求**: 5.1, 5.2, 5.3, 5.4

**描述**: 创建 Loguru 日志配置模块

**文件操作**:
- 创建 `common/logging_config.py`

**实现内容**:
```python
import sys
from loguru import logger

def setup_logging(env: str = "development"):
    """配置 Loguru 日志系统"""
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
            serialize=True,
        )
```

**验收标准**:
- 开发环境输出彩色日志
- 生产环境输出 JSON 格式
- 日志包含时间戳、级别、模块、函数、行号

---

- [ ] 7. 创建统一响应格式 (common/response_envelope.py)

**需求**: 6.1, 6.3, 6.4

**描述**: 创建统一的 API 响应格式定义

**文件操作**:
- 创建 `common/response_envelope.py`

**实现内容**:
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
    
    class Config:
        json_schema_extra = {
            "example": {
                "code": 200,
                "message": "success",
                "data": {"id": 1, "name": "example"}
            }
        }
```

**验收标准**:
- 支持泛型类型
- 包含 code、message、data、error 字段
- 提供示例

---

- [ ] 8. 创建响应包装中间件 (common/envelope_middleware.py)

**需求**: 6.2

**描述**: 创建自动包装 API 响应的中间件

**文件操作**:
- 创建 `common/envelope_middleware.py`

**实现内容**:
```python
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
import json

class ResponseEnvelopeMiddleware(BaseHTTPMiddleware):
    """自动包装 API 响应的中间件"""
    
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        
        # 跳过文档和静态文件
        if request.url.path in ["/docs", "/redoc", "/openapi.json"]:
            return response
        
        # 如果响应已经是包装格式，直接返回
        # 否则包装成统一格式
        
        return response
```

**验收标准**:
- 自动包装 API 响应
- 跳过文档和静态文件
- 不重复包装已包装的响应

---

- [ ] 9. 创建 JSON 工具 (common/json_utils.py)

**需求**: 通用工具

**描述**: 从 OneManage 复制 JSON 序列化工具

**文件操作**:
- 完整复制 `OneManage/common/json_utils.py` 到 `common/json_utils.py`

**验收标准**:
- 提供 JSON 序列化和反序列化工具
- 支持日期时间类型

---

- [ ] 10. 创建事件发布器 (events/publisher.py)

**需求**: 3.3

**描述**: 创建事件发布工具

**文件操作**:
- 创建 `events/publisher.py`

**实现内容**:
```python
from core.broker import broker
from loguru import logger
import json

async def publish_event(subject: str, event_data: dict):
    """发布事件到 NATS
    
    Args:
        subject: 事件主题（如 events.example.created）
        event_data: 事件数据
    """
    try:
        # 发布到 NATS
        await broker.client.publish(
            subject,
            json.dumps(event_data).encode()
        )
        logger.info(f"[Event] 发布事件: {subject}")
    except Exception as e:
        logger.error(f"[Event] 发布事件失败: {subject}, error={e}")
        raise
```

**验收标准**:
- 支持发布事件到 NATS
- 记录日志
- 异常处理

---

- [ ] 11. 创建事件 Schema (events/schemas.py)

**需求**: 3.3

**描述**: 创建事件 Schema 定义

**文件操作**:
- 创建 `events/schemas.py`

**实现内容**:
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

**验收标准**:
- 提供 BaseEvent 基类
- 提供示例事件 Schema

---

- [ ] 12. 创建事件主题定义 (events/subjects.py)

**需求**: 3.3

**描述**: 创建事件主题常量定义

**文件操作**:
- 创建 `events/subjects.py`

**实现内容**:
```python
"""NATS 事件主题定义"""

# 示例事件主题
EXAMPLE_CREATED = "events.example.created"
EXAMPLE_UPDATED = "events.example.updated"
EXAMPLE_DELETED = "events.example.deleted"
```

**验收标准**:
- 定义事件主题常量
- 遵循命名规范（events.{domain}.{action}）

---

## P2 - 工具脚本

- [ ] 13. 创建数据库连接测试脚本 (scripts/test_db_connection.py)

**需求**: 7.1, 7.2, 7.3, 7.4

**描述**: 创建数据库连接测试脚本

**文件操作**:
- 创建 `scripts/test_db_connection.py`

**实现内容**: 参考设计文档中的实现

**验收标准**:
- 测试数据库连接
- 执行简单查询获取版本
- 输出成功或失败信息
- 提供配置检查建议

---

- [ ] 14. 创建 OSS 连接测试脚本 (scripts/test_oss_connection.py)

**需求**: 7.5, 7.6, 7.7, 7.8

**描述**: 创建 OSS 连接测试脚本

**文件操作**:
- 创建 `scripts/test_oss_connection.py`

**实现内容**: 参考设计文档中的实现

**验收标准**:
- 测试 OSS 连接
- 列举存储桶
- 上传和删除测试文件
- 输出成功或失败信息

---

- [ ] 15. 创建 NATS 连接测试脚本 (scripts/test_nats_connection.py)

**需求**: 7.9, 7.10

**描述**: 创建 NATS 连接测试脚本

**文件操作**:
- 创建 `scripts/test_nats_connection.py`

**实现内容**: 参考设计文档中的实现

**验收标准**:
- 测试 NATS 连接
- 检查 TASKS 和 EVENTS 流是否存在
- 输出成功或失败信息

---

- [ ] 16. 创建健康检查脚本 (scripts/health_check.py)

**需求**: 7.11, 7.12

**描述**: 创建一键健康检查脚本

**文件操作**:
- 创建 `scripts/health_check.py`

**实现内容**: 参考设计文档中的实现

**验收标准**:
- 依次检查数据库、NATS、OSS
- 输出汇总报告
- 返回非零退出码如果有失败

---

- [ ] 17. 创建 JetStream 初始化脚本 (scripts/setup_jetstream.py)

**需求**: 3.1

**描述**: 从 OneManage 复制 JetStream 初始化脚本

**文件操作**:
- 复制 `OneManage/scripts/setup_jetstream.py` 到 `scripts/setup_jetstream.py`
- 根据模板项目调整流配置

**验收标准**:
- 创建 TASKS 流
- 创建 EVENTS 流
- 幂等操作（流已存在则跳过）

---

- [ ] 18. 创建项目初始化脚本 (scripts/init_project.py)

**需求**: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7

**描述**: 创建项目初始化脚本

**文件操作**:
- 创建 `scripts/init_project.py`

**实现内容**:
```python
import argparse
import secrets
from pathlib import Path
import re

def init_project(
    project_name: str,
    db_name: str,
    bucket: str | None = None,
    nats_stream: str | None = None,
):
    """初始化项目配置
    
    1. 复制 .env.example 到 .env.development
    2. 替换占位符
    3. 生成随机 SECRET_KEY
    4. 更新 pyproject.toml
    5. 输出下一步指引
    """
    # 实现逻辑
```

**验收标准**:
- 接受命令行参数
- 复制并修改配置文件
- 生成随机密钥
- 更新 pyproject.toml
- 输出操作指引

---

## P3 - 配置文件和示例

- [ ] 19. 创建配置文件模板 (.env.example)

**需求**: 1.2

**描述**: 创建环境变量配置模板

**文件操作**:
- 创建 `.env.example`

**实现内容**: 参考设计文档中的配置

**验收标准**:
- 包含所有必需的配置项
- 使用占位符
- 包含注释说明

---

- [ ] 20. 创建依赖管理文件 (pyproject.toml)

**需求**: 11.1, 11.2, 11.3

**描述**: 创建 PDM 依赖管理文件

**文件操作**:
- 创建 `pyproject.toml`

**实现内容**: 参考设计文档中的配置

**验收标准**:
- 包含所有核心依赖
- 配置 Black 和 Ruff
- 使用 Python 3.12

---

- [ ] 21. 创建 Alembic 配置 (alembic.ini)

**需求**: 9.1

**描述**: 创建数据库迁移配置文件

**文件操作**:
- 创建 `alembic.ini`

**实现内容**: 参考设计文档中的配置

**验收标准**:
- 配置迁移脚本位置
- 配置日志

---

- [ ] 22. 创建 Alembic 环境配置 (migrations/env.py)

**需求**: 9.2

**描述**: 创建 Alembic 环境配置

**文件操作**:
- 创建 `migrations/env.py`

**实现内容**: 参考设计文档中的实现

**验收标准**:
- 支持异步迁移
- 从 config 读取数据库 URL
- 自动发现模型

---

- [ ] 23. 创建迁移脚本模板 (migrations/script.py.mako)

**需求**: 9.3

**描述**: 从 OneManage 复制迁移脚本模板

**文件操作**:
- 复制 `OneManage/migrations/script.py.mako` 到 `migrations/script.py.mako`

**验收标准**:
- 提供标准的迁移脚本模板

---

- [ ] 24. 创建 .gitignore

**需求**: 12.1

**描述**: 创建 Git 忽略规则文件

**文件操作**:
- 创建 `.gitignore`

**实现内容**: 参考设计文档中的配置

**验收标准**:
- 忽略 Python 缓存
- 忽略环境变量文件
- 忽略 IDE 配置

---

- [ ] 25. 创建示例数据模型 (features/example/models.py)

**需求**: 10.1, 10.2

**描述**: 创建示例 SQLModel 数据模型

**文件操作**:
- 创建 `features/example/` 目录
- 创建 `features/example/__init__.py`
- 创建 `features/example/models.py`

**实现内容**: 参考设计文档中的示例

**验收标准**:
- 使用 SQLModel 定义模型
- 包含基本字段和时间戳

---

- [ ] 26. 创建示例 Schema (features/example/schemas.py)

**需求**: 10.3

**描述**: 创建示例 Pydantic Schema

**文件操作**:
- 创建 `features/example/schemas.py`

**实现内容**: 参考设计文档中的示例

**验收标准**:
- 定义 Create、Update、Response Schema
- 使用 Pydantic BaseModel

---

- [ ] 27. 创建示例路由 (features/example/routers.py)

**需求**: 10.4

**描述**: 创建示例 FastAPI 路由

**文件操作**:
- 创建 `features/example/routers.py`

**实现内容**: 参考设计文档中的示例

**验收标准**:
- 实现 CRUD 接口
- 使用依赖注入
- 使用 Schema 验证

---

- [ ] 28. 创建示例业务逻辑 (features/example/services.py)

**需求**: 10.5

**描述**: 创建示例业务逻辑层

**文件操作**:
- 创建 `features/example/services.py`

**实现内容**: 参考设计文档中的示例

**验收标准**:
- 实现 CRUD 操作
- 发布事件
- 异常处理

---

- [ ] 29. 创建示例异步任务 (features/example/tasks.py)

**需求**: 10.6

**描述**: 创建示例 TaskIQ 异步任务

**文件操作**:
- 创建 `features/example/tasks.py`

**实现内容**: 参考设计文档中的示例

**验收标准**:
- 定义命令任务（@broker.task）
- 定义事件处理器（@events_broker.task）
- 展示任务调用方式

---

- [ ] 30. 创建应用入口 (main.py)

**需求**: 所有需求

**描述**: 创建 FastAPI 应用入口文件

**文件操作**:
- 创建 `main.py`

**实现内容**: 参考设计文档中的实现

**验收标准**:
- 配置日志
- 注册中间件
- 注册路由
- 配置生命周期
- 提供健康检查端点

---

## 任务执行顺序建议

### 第一阶段：核心基础（P0）
1. 任务 1: 创建目录结构
2. 任务 2: 创建 config.py ⭐ **最高优先级**
3. 任务 3: 创建 database.py ⭐ **最高优先级**

### 第二阶段：基础组件（P1）
4. 任务 4: 创建 broker.py
5. 任务 5: 创建 oss.py
6. 任务 6: 创建 logging_config.py
7. 任务 7-9: 创建响应格式相关
8. 任务 10-12: 创建事件相关

### 第三阶段：工具脚本（P2）
9. 任务 13-17: 创建验证脚本
10. 任务 18: 创建初始化脚本

### 第四阶段：配置和示例（P3）
11. 任务 19-24: 创建配置文件
12. 任务 25-29: 创建示例模块
13. 任务 30: 创建应用入口

---

## 验证清单

完成所有任务后，执行以下验证：

- [ ] 运行 `pdm install` 安装依赖
- [ ] 运行 `pdm run python scripts/test_db_connection.py` 测试数据库
- [ ] 运行 `pdm run python scripts/test_nats_connection.py` 测试 NATS
- [ ] 运行 `pdm run python scripts/test_oss_connection.py` 测试 OSS
- [ ] 运行 `pdm run python scripts/health_check.py` 一键健康检查
- [ ] 运行 `pdm run python scripts/setup_jetstream.py` 初始化 JetStream
- [ ] 运行 `pdm run alembic revision --autogenerate -m "Initial"` 生成迁移
- [ ] 运行 `pdm run alembic upgrade head` 应用迁移
- [ ] 运行 `pdm run uvicorn main:app --reload` 启动 API
- [ ] 访问 `http://localhost:8000/docs` 查看文档
- [ ] 测试示例 API 接口
