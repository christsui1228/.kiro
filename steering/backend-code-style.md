---
inclusion: fileMatch
fileMatchPattern: '*.py'
---

# 代码风格规范

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

**推荐使用 `common/base.py` 中的 `TimestampModel` 基类**，它提供了标准的时间戳字段。

**重要：如果不使用 `common/base.py`，时间戳字段必须使用 `lambda: datetime.now(timezone.utc)` 写法！**

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
    """时间戳 Mixin，提供 UTC 时间戳字段
    
    注意：必须使用 lambda: datetime.now(timezone.utc) 确保每次调用都生成新的时间戳
    """
    created_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc),  # 必须使用 lambda
        nullable=False,
        sa_type=DateTime(timezone=True),
        sa_column_kwargs={"server_default": func.now()},
        description="创建时间（UTC）"
    )
    updated_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc),  # 必须使用 lambda
        nullable=False,
        sa_type=DateTime(timezone=True),
        sa_column_kwargs={"server_default": func.now(), "onupdate": func.now()},
        description="更新时间（UTC）"
    )

class TimestampModel(BaseModel, TimestampMixin):
    """基础时间戳模型，所有需要时间戳的模型都继承此类"""
    pass
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
from common.exceptions import ValidationError
```

## 时间戳规范

### UTC 时间戳标准
**必须使用 UTC 时间戳**: `lambda: datetime.now(timezone.utc)`

**关键要点**：
1. **优先使用** `common/base.py` 中的 `TimestampModel` 基类
2. **如果不使用基类**，时间戳字段必须按照 `lambda: datetime.now(timezone.utc)` 写法
3. **禁止使用** `datetime.now` 或 `datetime.now()` 作为默认值

```python
from datetime import datetime, timezone
from sqlalchemy import DateTime

# ✅ 推荐方式1：继承 TimestampModel（最佳实践）
from common.base import TimestampModel

class Order(TimestampModel, table=True):
    __tablename__ = "orders"
    # created_at 和 updated_at 自动继承

# ✅ 推荐方式2：不使用基类时，必须用 lambda
class CustomModel(SQLModel, table=True):
    created_at: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc),  # 必须使用 lambda
        sa_type=DateTime(timezone=True)
    )

# ❌ 错误写法1：没有时区信息
created_at: datetime = Field(default_factory=datetime.now)

# ❌ 错误写法2：本地时间
created_at: datetime = Field(default_factory=lambda: datetime.now())

# ❌ 错误写法3：直接调用函数（所有实例共享同一个时间戳）
created_at: datetime = Field(default=datetime.now(timezone.utc))
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
