---
inclusion: fileMatch
fileMatchPattern: 'features/**/*.py'
---

# 后端架构规范

## 项目结构规范

### 根目录结构
保持根目录尽可能干净，按功能分类组织：

```
OneManage/
├── features/              # 业务功能模块
├── common/               # 共享组件
├── core/                 # 核心配置
├── events/               # 事件系统
├── tests/                # 测试代码
├── scripts/              # 初始化脚本
├── logs/                 # 日志文件 (运行时生成)
├── alembic/              # 数据库迁移
├── main.py               # 应用入口
├── pyproject.toml        # 项目配置
├── Dockerfile            # 容器配置
├── docker-compose.yml    # 容器编排
├── .env.example          # 环境变量模板
└── README.md             # 项目文档
```

## FastAPI 架构模式

### Feature-Driven Development (FDD)
与 Feature-Driven 对应的是 **Layer-Driven** 或 **技术分层架构**：

- **Feature-Driven** (推荐): 按业务功能组织代码
  ```
  features/
  ├── auth/           # 认证功能
  ├── orders/         # 订单功能  
  ├── users/          # 用户功能
  └── payments/       # 支付功能
  ```

- **Layer-Driven** (避免): 按技术层次组织代码
  ```
  src/
  ├── models/         # 所有模型
  ├── services/       # 所有服务
  ├── controllers/    # 所有控制器
  └── repositories/   # 所有仓库
  ```

**Feature-Driven 的优势**:
- 高内聚：相关功能代码集中
- 低耦合：功能间依赖清晰
- 易维护：修改功能只需关注一个目录
- 团队协作：不同团队负责不同功能模块

### 功能模块结构
每个功能模块遵循统一的目录结构：

```
features/{module_name}/
├── __init__.py
├── models.py          # SQLModel 数据模型 + API 请求/响应模型 (需要持久化时)
├── schemas.py         # API 请求/响应模型 (仅当不需要持久化时)
├── crud.py            # 数据库操作层
├── services.py        # 业务逻辑层
├── routers.py         # API 路由层
├── tasks.py           # TaskIQ 异步任务
├── exceptions.py      # 自定义异常 (可选)
└── utils.py           # 工具函数 (可选)
```

**数据模型文件选择规范**：
- **有数据库表**: 使用 `models.py`，同时包含 SQLModel 数据库模型和 API 请求/响应模型
- **无数据库表**: 使用 `schemas.py`，仅包含 API 请求/响应模型
- **符合 SQLModel 最佳实践**: 将数据库模型和 API 模型放在同一文件中，便于共享字段定义

### 共享组件组织
```
common/
├── base.py                    # 基础模型和时间戳 Mixin
├── exceptions.py              # 通用异常
├── utils.py                   # 通用工具函数
├── validators.py              # 通用验证器
├── envelope_middleware.py     # 响应封装中间件
└── response_envelope.py       # 统一响应格式

core/
├── config.py              # 配置管理
├── database_async.py      # 异步数据库连接和会话
├── database_sync.py       # 同步数据库连接和会话
├── broker.py              # TaskIQ 配置
└── security.py            # 安全相关

### 数据库会话管理

项目使用两种数据库会话方式：

#### 异步会话 (`database_async.py`)
用于 FastAPI 路由和异步任务：
```python
from core.database_async import get_session, get_session_context

# 方式 1: FastAPI 依赖注入
@router.get("/users")
async def list_users(db: AsyncSession = Depends(get_session)):
    users = await db.execute(select(User))
    return users.scalars().all()

# 方式 2: 上下文管理器（用于事件处理器、后台任务）
async with get_session_context() as db:
    user = await db.get(User, user_id)
    await db.commit()
```

#### 同步会话 (`database_sync.py`)
用于数据分析和批量处理（配合 Polars）：
```python
from core.database_sync import get_sync_session_context

# 用于 Polars + ConnectorX 数据分析
with get_sync_session_context() as db:
    # 执行同步数据库操作
    result = db.execute(select(User))
    users = result.scalars().all()
```

**使用场景**：
- ✅ **异步会话**: FastAPI 路由、TaskIQ 任务、事件处理器
- ✅ **同步会话**: 数据分析脚本、批量 ETL 任务、Polars 数据处理

events/
├── subjects.py        # 事件主题定义
├── schemas.py         # 事件数据模型 (事件系统不涉及数据库，使用 schemas.py)
├── publisher.py       # 事件发布器
└── consumer.py        # 事件消费者
```

### 测试代码组织
```
tests/
├── conftest.py              # pytest 配置和 fixtures
├── test_auth/               # 认证功能测试
│   ├── test_models.py
│   ├── test_services.py
│   └── test_routers.py
├── test_orders/             # 订单功能测试
├── test_common/             # 共享组件测试
├── integration/             # 集成测试
└── performance/             # 性能测试
```

### 脚本文件组织
```
scripts/
├── init_db.py              # 数据库初始化
├── seed_data.py            # 测试数据生成
├── migrate.py              # 数据迁移脚本
├── backup.py               # 数据备份脚本
├── run_event_consumer.py   # 事件消费者启动
└── deploy.py               # 部署脚本
```

## 分层架构职责

### 1. Models Layer (`models.py`)

**场景 A: 需要数据库持久化（推荐使用 SQLModel 最佳实践）**

将数据库模型和 API 模型放在同一个 `models.py` 文件中：

```python
# features/users/models.py
from sqlmodel import SQLModel, Field
from pydantic import BaseModel
import uuid

# ========== 数据库模型 ==========
class User(SQLModel, table=True):
    """用户数据库表模型"""
    __tablename__ = "users"
    
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    name: str = Field(max_length=100)
    email: str | None = Field(default=None)
    created_at: datetime = Field(default_factory=datetime.utcnow)

# ========== API 请求/响应模型 ==========
class UserCreateRequest(BaseModel):
    """创建用户请求模型"""
    name: str
    email: str | None = None

class UserUpdateRequest(BaseModel):
    """更新用户请求模型"""
    name: str | None = None
    email: str | None = None

class UserResponse(BaseModel):
    """用户响应模型"""
    id: uuid.UUID
    name: str
    email: str | None
    created_at: datetime
    
    class Config:
        from_attributes = True
```

**场景 B: 不需要数据库持久化（使用 schemas.py）**

仅有 API 请求/响应模型，不涉及数据库：

```python
# features/analysis/schemas.py
from pydantic import BaseModel

class AnalysisRequest(BaseModel):
    """分析请求模型（不需要持久化）"""
    data: list[dict]
    analysis_type: str

class AnalysisResponse(BaseModel):
    """分析结果响应模型（不需要持久化）"""
    result: dict
    summary: str
```

**选择规则**：
- ✅ **有数据库表**: 使用 `models.py`，包含数据库模型 + API 模型
- ✅ **无数据库表**: 使用 `schemas.py`，仅包含 API 模型
- ✅ **符合 SQLModel 最佳实践**: 共享字段定义，减少重复代码

**实际项目示例**：
```
features/
├── users/                    # 用户模块（需要持久化）
│   ├── models.py            # ✅ 包含 User 表 + UserCreateRequest + UserResponse
│   ├── crud.py
│   ├── services.py
│   └── routers.py
│
├── business_analysis/        # 业务分析模块（不需要持久化）
│   ├── schemas.py           # ✅ 仅包含 AnalysisRequest + AnalysisResponse
│   ├── analysis.py          # 分析逻辑
│   └── routers.py
│
└── data_import/              # 数据导入模块（需要持久化）
    ├── models.py            # ✅ 包含 ImportRecord 表 + ImportRequest + ImportResponse
    ├── crud.py
    ├── processor.py
    └── routers.py
```

### 2. CRUD Layer (`crud.py`)
```python
# 数据库操作，Repository 模式
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
import uuid
from .models import User  # 从 models.py 导入数据库模型

class UserCRUD:
    @staticmethod
    async def create(db: AsyncSession, user_data: dict) -> User:
        user = User(**user_data)
        db.add(user)
        await db.commit()
        await db.refresh(user)
        return user
    
    @staticmethod
    async def get_by_id(db: AsyncSession, user_id: uuid.UUID) -> User | None:
        result = await db.execute(select(User).where(User.id == user_id))
        return result.scalar_one_or_none()
```

### 3. Services Layer (`services.py`)
```python
# 业务逻辑层
from sqlalchemy.ext.asyncio import AsyncSession
from .crud import UserCRUD
from .models import UserCreateRequest, UserResponse  # 从 models.py 导入 API 模型

class UserService:
    def __init__(self, db: AsyncSession):
        self.db = db
        self.crud = UserCRUD()
    
    async def create_user(self, user_data: UserCreateRequest) -> UserResponse:
        # 业务逻辑验证
        if await self._email_exists(user_data.email):
            raise ValueError("Email already exists")
        
        # 创建用户
        user = await self.crud.create(self.db, user_data.model_dump())
        return UserResponse.model_validate(user)
    
    async def _email_exists(self, email: str) -> bool:
        # 私有业务逻辑方法
        pass
```

### 4. Routers Layer (`routers.py`)
```python
# API 路由层，Controller
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from core.database_async import get_session
from .services import UserService
from .models import UserCreateRequest, UserResponse  # 从 models.py 导入 API 模型

router = APIRouter(prefix="/users", tags=["Users"])

@router.post("/", response_model=UserResponse)
async def create_user(
    user_data: UserCreateRequest,
    db: AsyncSession = Depends(get_session)
):
    service = UserService(db)
    try:
        return await service.create_user(user_data)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
```

### 5. Tasks Layer (`tasks.py`)

**双流架构：命令任务（TASKS流）+ 事件处理器（EVENTS流）**

```python
# features/*/tasks.py
from core.broker import broker
from events.publisher import publish_event
from loguru import logger

# ========== 命令任务（TASKS流）==========
@broker.task
async def task_send_welcome_email(user_id: str, email: str) -> dict:
    """发送欢迎邮件异步任务（命令任务）
    
    注意：使用 @broker.task 装饰器
    由 TaskIQ Worker 处理
    
    Args:
        user_id: 用户ID
        email: 邮箱地址
        
    Returns:
        dict: 处理结果
    """
    try:
        # 发布开始事件到 EVENTS 流
        await publish_event("user.email.started", {
            "user_id": user_id,
            "email": email,
            "status": "started"
        })
        
        # 执行业务逻辑
        await send_email(email, "Welcome!")
        
        # 发布完成事件到 EVENTS 流
        await publish_event("user.email.completed", {
            "user_id": user_id,
            "email": email,
            "status": "completed"
        })
        
        return {"user_id": user_id, "status": "sent"}
        
    except Exception as e:
        # 发布失败事件
        await publish_event("user.email.failed", {
            "user_id": user_id,
            "email": email,
            "error": str(e),
            "status": "failed"
        })
        logger.error(f"Failed to send welcome email: {e}")
        raise

# ========== 事件处理器（EVENTS流）==========
async def on_user_email_completed(raw_payload: dict) -> None:
    """监听邮件发送完成事件（事件处理器）
    
    注意：
    1. 不使用 @broker.task 装饰器
    2. 参数是 raw_payload (dict)，不是 Pydantic 模型
    3. 在事件消费者中注册此处理器
    4. 由独立的事件消费者服务处理
    
    Args:
        raw_payload: 纯 JSON 格式的事件数据
    """
    try:
        logger.info(f"收到邮件发送完成事件: user_id={raw_payload.get('user_id')}")
        
        # 更新数据库状态
        from core.database_async import get_session_context
        async with get_session_context() as db:
            # 更新用户邮件发送状态
            await update_user_email_status(db, raw_payload["user_id"], "sent")
            await db.commit()
            
        logger.info("事件处理完成")
        
    except Exception as e:
        logger.error(f"事件处理失败: {e}")
        # 不重新抛出异常，避免影响其他监听器
```

**事件消费者注册（scripts/run_event_consumer.py）**：
```python
from events.consumer import register_event_handler
from features.users.tasks import on_user_email_completed

async def setup_event_consumers():
    """注册所有事件处理器"""
    
    # 注册事件处理器
    register_event_handler(
        "events.user.email.completed",  # Subject
        on_user_email_completed         # 处理器函数
    )
```

## 架构最佳实践

### 依赖方向
```
Routers → Services → CRUD → Models
   ↓         ↓
Schemas   Tasks/Events
```

### 职责分离原则
- **Models**: 只定义数据结构，不包含业务逻辑
- **CRUD**: 只负责数据库操作，不包含业务逻辑
- **Services**: 包含所有业务逻辑和验证
- **Routers**: 只负责 HTTP 请求/响应处理
- **Tasks**: 异步任务和事件处理

### 跨模块通信
- **同步调用**: 通过 Service 层直接调用
- **异步任务**: 通过 TaskIQ 命令任务
- **事件通知**: 通过 NATS 事件流
