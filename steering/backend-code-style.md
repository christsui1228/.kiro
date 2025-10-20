---
inclusion: fileMatch
fileMatchPattern: '*.py'
---

# 代码风格规范

## 语言和交流规范
- **所有对话**: 必须使用简体中文
- **代码注释**: 必须使用简体中文
- **文档编写**: 必须使用简体中文
- **变量函数名**: 使用英文，但注释用简体中文

## 类型注解规范

### 联合类型语法
**优先使用现代联合类型语法** (Python 3.10+)

```python
# ✅ 推荐：使用 | None 语法
user_id: uuid.UUID | None = Field(default=None)
name: str | None = Field(default=None)
age: int | None = Field(default=None)
tags: list[str] | None = Field(default=None)

# ❌ 避免：使用 Union 或 Optional
from typing import Union, Optional
user_id: Union[uuid.UUID, None] = Field(default=None)
name: Optional[str] = Field(default=None)
```

### SQLModel 字段定义规范
```python
import uuid
from datetime import datetime, timezone
from sqlmodel import SQLModel, Field
from sqlalchemy import DateTime, func
from common.base import TimestampModel

class User(TimestampModel, table=True):
    __tablename__ = "users"
    
    # 主键使用 UUID
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True, index=True)
    
    # 外键关联
    shop_id: uuid.UUID | None = Field(
        default=None, 
        foreign_key="shop_configs.shop_id", 
        index=True
    )
    
    # 基础字段
    name: str = Field(max_length=100, index=True)
    email: str | None = Field(default=None, max_length=255, index=True)
    phone: str | None = Field(default=None, max_length=20)
    
    # 数值字段
    age: int | None = Field(default=None, ge=0, le=150)
    balance: float | None = Field(default=None, ge=0)
    
    # 枚举字段
    status: UserStatus = Field(default=UserStatus.ACTIVE, index=True)
    
    # JSON 字段
    metadata: dict | None = Field(default=None)
    tags: list[str] | None = Field(default=None)
    
    # 时间戳字段通过 TimestampModel 自动继承：
    # created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    # updated_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
```

### 时间戳基类规范 (`common/base.py`)
```python
from datetime import datetime, timezone
from sqlmodel import SQLModel, Field
from sqlalchemy import DateTime, func

class BaseModel(SQLModel):
    """基础模型，包含通用字段和配置"""
    id: int | None = Field(
        default=None,
        primary_key=True,
        index=True,
        nullable=False,
        sa_column_kwargs={"autoincrement": True}
    )
    
    model_config = {
        "from_attributes": True,
        "arbitrary_types_allowed": True,
        "json_encoders": {
            datetime: lambda dt: dt.isoformat()
        }
    }

class TimestampMixin:
    """时间戳 Mixin，提供 UTC 时间戳字段"""
    created_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc),
        nullable=False,
        sa_type=DateTime(timezone=True),
        sa_column_kwargs={"server_default": func.now()},
        description="创建时间（UTC）"
    )
    updated_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc),
        nullable=False,
        sa_type=DateTime(timezone=True),
        sa_column_kwargs={"server_default": func.now(), "onupdate": func.now()},
        description="更新时间（UTC）"
    )

class TimestampModel(BaseModel, TimestampMixin):
    """基础时间戳模型，所有需要时间戳的模型都继承此类"""
    pass
```

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

### FastAPI 架构模式

#### Feature-Driven Development (FDD)
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
├── models.py          # SQLModel 数据模型
├── schemas.py         # msgspec/Pydantic API 模型
├── crud.py            # 数据库操作层
├── services.py        # 业务逻辑层
├── routers.py         # API 路由层
├── tasks.py           # TaskIQ 异步任务
├── exceptions.py      # 自定义异常 (可选)
└── utils.py           # 工具函数 (可选)
```

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
├── config.py          # 配置管理
├── database.py        # 数据库连接
├── broker.py          # TaskIQ 配置
└── security.py        # 安全相关

events/
├── subjects.py        # 事件主题定义
├── schemas.py         # 事件数据模型
├── publisher.py       # 事件发布器
└── subscriber.py      # 事件订阅器
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
└── deploy.py               # 部署脚本
```

### 分层架构职责

#### 1. Models Layer (`models.py`)
```python
# SQLModel 数据库表定义
from sqlmodel import SQLModel, Field
import uuid

class User(SQLModel, table=True):
    __tablename__ = "users"
    
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    name: str = Field(max_length=100)
    email: str | None = Field(default=None)
```

#### 2. Schemas Layer (`schemas.py`)
```python
# API 请求/响应模型
from pydantic import BaseModel
import uuid

class UserCreateRequest(BaseModel):
    name: str
    email: str | None = None

class UserResponse(BaseModel):
    id: uuid.UUID
    name: str
    email: str | None
    
    class Config:
        from_attributes = True
```

#### 3. CRUD Layer (`crud.py`)
```python
# 数据库操作，Repository 模式
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
import uuid

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

#### 4. Services Layer (`services.py`)
```python
# 业务逻辑层
from sqlalchemy.ext.asyncio import AsyncSession
from .crud import UserCRUD
from .schemas import UserCreateRequest, UserResponse

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

#### 5. Routers Layer (`routers.py`)
```python
# API 路由层，Controller
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from core.database import get_session
from .services import UserService
from .schemas import UserCreateRequest, UserResponse

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

#### 6. Tasks Layer (`tasks.py`)
```python
# TaskIQ 异步任务和事件监听
from core.broker import broker
from events.publisher import publish_event
from events.subjects import USER_CREATED

@broker.task
async def send_welcome_email(user_id: str, email: str) -> None:
    """发送欢迎邮件异步任务"""
    try:
        # 发送邮件逻辑
        await send_email(email, "Welcome!")
        
        # 发布完成事件
        await publish_event(USER_CREATED, {
            "user_id": user_id,
            "email": email,
            "status": "email_sent"
        })
    except Exception as e:
        logger.error(f"Failed to send welcome email: {e}")
        raise
```

## 命名规范

### 文件和目录命名
- **模块目录**: 使用下划线分隔 (`data_import`, `image_generator`)
- **Python 文件**: 使用下划线分隔 (`user_service.py`, `order_models.py`)
- **类名**: 使用 PascalCase (`UserService`, `OrderModel`)
- **函数/变量**: 使用 snake_case (`create_user`, `user_id`)

### 数据库命名
```python
class User(SQLModel, table=True):
    __tablename__ = "users"  # 表名：复数形式，下划线分隔
    
    # 字段名：下划线分隔
    user_id: uuid.UUID = Field(primary_key=True)
    first_name: str
    created_at: datetime
    
    # 外键：{关联表单数}_id
    shop_id: uuid.UUID | None = Field(foreign_key="shops.id")
```

### API 路由命名
```python
# RESTful 风格
router = APIRouter(prefix="/api/users", tags=["Users"])

@router.get("/")                    # GET /api/users
@router.post("/")                   # POST /api/users
@router.get("/{user_id}")          # GET /api/users/{user_id}
@router.put("/{user_id}")          # PUT /api/users/{user_id}
@router.delete("/{user_id}")       # DELETE /api/users/{user_id}

# 嵌套资源
@router.get("/{user_id}/orders")   # GET /api/users/{user_id}/orders
```

## 导入规范

### 导入顺序
```python
# 1. 标准库
import uuid
from datetime import datetime
from typing import Any

# 2. 第三方库
from fastapi import APIRouter, Depends
from sqlmodel import SQLModel, Field
from pydantic import BaseModel

# 3. 本地导入
from core.database import get_session
from core.config import settings
from .models import User
from .schemas import UserResponse
```

### 相对导入
```python
# 在同一模块内使用相对导入
from .models import User
from .crud import UserCRUD
from .schemas import UserResponse

# 跨模块使用绝对导入
from features.auth.models import User
from features.common.exceptions import ValidationError
```

## 错误处理规范

### 异常定义
```python
# features/{module}/exceptions.py
class UserNotFoundError(Exception):
    """用户不存在异常"""
    pass

class EmailAlreadyExistsError(Exception):
    """邮箱已存在异常"""
    pass
```

### 异常处理
```python
# 在 Service 层抛出业务异常
async def get_user(self, user_id: uuid.UUID) -> UserResponse:
    user = await self.crud.get_by_id(self.db, user_id)
    if not user:
        raise UserNotFoundError(f"User {user_id} not found")
    return UserResponse.model_validate(user)

# 在 Router 层转换为 HTTP 异常
@router.get("/{user_id}")
async def get_user(user_id: uuid.UUID, db: AsyncSession = Depends(get_session)):
    service = UserService(db)
    try:
        return await service.get_user(user_id)
    except UserNotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))
```

## 时间戳规范

### UTC 时间戳标准
**必须使用 UTC 时间戳**: `lambda: datetime.now(timezone.utc)`

```python
from datetime import datetime, timezone

# ✅ 推荐：UTC 时间戳
created_at: datetime = Field(
    default_factory=lambda: datetime.now(timezone.utc),
    sa_type=DateTime(timezone=True)
)

# ❌ 避免：本地时间
created_at: datetime = Field(default_factory=datetime.now)  # 没有时区信息
created_at: datetime = Field(default_factory=lambda: datetime.now())  # 本地时间
```

### 时间戳继承模式
```python
# 所有需要时间戳的模型都继承 TimestampModel
from common.base import TimestampModel

class Order(TimestampModel, table=True):
    __tablename__ = "orders"
    
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    amount: float
    # created_at 和 updated_at 自动继承
```

### 手动时间戳字段
```python
# 业务特定的时间字段也使用 UTC
class Order(TimestampModel, table=True):
    # 继承的时间戳
    # created_at: datetime (UTC)
    # updated_at: datetime (UTC)
    
    # 业务时间字段
    order_date: datetime | None = Field(
        default_factory=lambda: datetime.now(timezone.utc),
        sa_type=DateTime(timezone=True),
        description="订单时间（UTC）"
    )
    completed_at: datetime | None = Field(
        default=None,
        sa_type=DateTime(timezone=True),
        description="完成时间（UTC）"
    )
```

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
# 实际使用示例
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

### Polars vs Pandas 使用场景
- **Polars 优势**: 数据清洗、转换、聚合分析
- **Pandas 优势**: 数据库写入、生态兼容性
- **最佳实践**: Polars 处理 + pandas 写入

## 序列化规范

### msgspec vs Pydantic 选择策略
- **msgspec**: 简单数据结构，追求性能
- **Pydantic**: 复杂验证逻辑，功能完整

```python
# ✅ 使用 msgspec (简单场景)
import msgspec

class UserCreateRequest(msgspec.Struct):
    name: str
    email: str | None = None
    age: int | None = None

class UserListResponse(msgspec.Struct):
    users: list[UserResponse]
    total: int
    page: int

# ✅ 使用 Pydantic (复杂场景)
from pydantic import BaseModel, validator, Field

class ComplexUserRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    email: str = Field(..., regex=r'^[\w\.-]+@[\w\.-]+\.\w+$')
    password: str = Field(..., min_length=8)
    
    @validator('password')
    def validate_password(cls, v):
        if not any(c.isupper() for c in v):
            raise ValueError('Password must contain uppercase letter')
        return v
    
    class Config:
        schema_extra = {
            "example": {
                "name": "John Doe",
                "email": "john@example.com",
                "password": "SecurePass123"
            }
        }
```

### 响应封装使用
```python
from common.response_envelope import success, ResponseEnvelope

# 在 Router 中使用
@router.get("/users", response_model=ResponseEnvelope[list[UserResponse]])
async def get_users():
    users = await user_service.get_all()
    return success(users)  # 自动封装

# 错误响应
@router.post("/users")
async def create_user(user_data: UserCreateRequest):
    try:
        user = await user_service.create(user_data)
        return success(user)
    except ValueError as e:
        return ResponseEnvelope(code=400, message=str(e), data=None)
```

## 日志规范

### loguru 结构化日志
```python
from loguru import logger

# 业务操作日志 (结构化)
async def create_user(self, user_data: UserCreateRequest) -> UserResponse:
    logger.info("Creating user", email=user_data.email, name=user_data.name)
    
    try:
        user = await self.crud.create(self.db, user_data)
        logger.info("User created successfully", 
                   user_id=str(user.id), 
                   email=user.email,
                   operation="user_creation")
        return UserResponse.from_orm(user)
    except Exception as e:
        logger.error("Failed to create user", 
                    email=user_data.email,
                    error=str(e),
                    error_type=type(e).__name__)
        raise

# 性能日志
import time
from functools import wraps

def log_performance(operation: str):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            start_time = time.time()
            try:
                result = await func(*args, **kwargs)
                duration = time.time() - start_time
                logger.info("Operation completed",
                           operation=operation,
                           duration=round(duration, 3),
                           success=True)
                return result
            except Exception as e:
                duration = time.time() - start_time
                logger.error("Operation failed",
                            operation=operation,
                            duration=round(duration, 3),
                            error=str(e),
                            success=False)
                raise
        return wrapper
    return decorator

# 使用示例
@log_performance("user_creation")
async def create_user_with_logging(user_data: UserCreateRequest):
    # 业务逻辑
    pass
```