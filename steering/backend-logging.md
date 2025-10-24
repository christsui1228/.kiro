---
inclusion: fileMatch
fileMatchPattern: '*.py'
---

# 日志规范

## loguru 结构化日志

### 基础日志配置

```python
from loguru import logger
import sys

# 配置日志输出
logger.remove()  # 移除默认处理器

# 控制台输出（开发环境）
logger.add(
    sys.stdout,
    format="<green>{time:YYYY-MM-DD HH:mm:ss}</green> | <level>{level: <8}</level> | <cyan>{name}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> - <level>{message}</level>",
    level="DEBUG"
)

# 文件输出（生产环境）
logger.add(
    "logs/app_{time:YYYY-MM-DD}.log",
    rotation="00:00",  # 每天午夜轮转
    retention="30 days",  # 保留30天
    compression="zip",  # 压缩旧日志
    format="{time:YYYY-MM-DD HH:mm:ss} | {level: <8} | {name}:{function}:{line} - {message}",
    level="INFO"
)

# 错误日志单独记录
logger.add(
    "logs/error_{time:YYYY-MM-DD}.log",
    rotation="00:00",
    retention="90 days",
    compression="zip",
    level="ERROR"
)
```

### 业务操作日志

```python
from loguru import logger

async def create_user(self, user_data: UserCreateRequest) -> UserResponse:
    """创建用户 - 结构化日志示例"""
    
    # 记录操作开始
    logger.info(
        "Creating user",
        email=user_data.email,
        name=user_data.name,
        operation="user_creation"
    )
    
    try:
        user = await self.crud.create(self.db, user_data)
        
        # 记录操作成功
        logger.info(
            "User created successfully", 
            user_id=str(user.id), 
            email=user.email,
            operation="user_creation",
            status="success"
        )
        
        return UserResponse.model_validate(user)
        
    except Exception as e:
        # 记录操作失败
        logger.error(
            "Failed to create user", 
            email=user_data.email,
            error=str(e),
            error_type=type(e).__name__,
            operation="user_creation",
            status="failed"
        )
        raise
```

### 性能日志装饰器

```python
import time
from functools import wraps
from loguru import logger

def log_performance(operation: str):
    """性能日志装饰器"""
    def decorator(func):
        @wraps(func)
        async def async_wrapper(*args, **kwargs):
            start_time = time.time()
            try:
                result = await func(*args, **kwargs)
                duration = time.time() - start_time
                
                logger.info(
                    "Operation completed",
                    operation=operation,
                    duration=round(duration, 3),
                    success=True
                )
                
                return result
                
            except Exception as e:
                duration = time.time() - start_time
                
                logger.error(
                    "Operation failed",
                    operation=operation,
                    duration=round(duration, 3),
                    error=str(e),
                    error_type=type(e).__name__,
                    success=False
                )
                raise
                
        @wraps(func)
        def sync_wrapper(*args, **kwargs):
            start_time = time.time()
            try:
                result = func(*args, **kwargs)
                duration = time.time() - start_time
                
                logger.info(
                    "Operation completed",
                    operation=operation,
                    duration=round(duration, 3),
                    success=True
                )
                
                return result
                
            except Exception as e:
                duration = time.time() - start_time
                
                logger.error(
                    "Operation failed",
                    operation=operation,
                    duration=round(duration, 3),
                    error=str(e),
                    error_type=type(e).__name__,
                    success=False
                )
                raise
        
        # 根据函数类型返回对应的包装器
        import asyncio
        if asyncio.iscoroutinefunction(func):
            return async_wrapper
        else:
            return sync_wrapper
            
    return decorator

# 使用示例
@log_performance("user_creation")
async def create_user_with_logging(user_data: UserCreateRequest):
    # 业务逻辑
    pass

@log_performance("data_processing")
def process_data(data: dict):
    # 同步业务逻辑
    pass
```

### API 请求日志中间件

```python
from fastapi import Request
from loguru import logger
import time

async def log_requests(request: Request, call_next):
    """记录所有 API 请求"""
    
    start_time = time.time()
    
    # 记录请求信息
    logger.info(
        "API request started",
        method=request.method,
        path=request.url.path,
        client_ip=request.client.host if request.client else None,
        user_agent=request.headers.get("user-agent")
    )
    
    try:
        response = await call_next(request)
        duration = time.time() - start_time
        
        # 记录响应信息
        logger.info(
            "API request completed",
            method=request.method,
            path=request.url.path,
            status_code=response.status_code,
            duration=round(duration, 3)
        )
        
        return response
        
    except Exception as e:
        duration = time.time() - start_time
        
        logger.error(
            "API request failed",
            method=request.method,
            path=request.url.path,
            error=str(e),
            error_type=type(e).__name__,
            duration=round(duration, 3)
        )
        raise

# 在 main.py 中注册中间件
app.middleware("http")(log_requests)
```

### 数据库操作日志

```python
from loguru import logger

class UserCRUD:
    @staticmethod
    async def create(db: AsyncSession, user_data: dict) -> User:
        """创建用户 - 带数据库操作日志"""
        
        logger.debug(
            "Database operation: create user",
            table="users",
            operation="INSERT",
            data=user_data
        )
        
        try:
            user = User(**user_data)
            db.add(user)
            await db.commit()
            await db.refresh(user)
            
            logger.debug(
                "Database operation completed",
                table="users",
                operation="INSERT",
                user_id=str(user.id)
            )
            
            return user
            
        except Exception as e:
            logger.error(
                "Database operation failed",
                table="users",
                operation="INSERT",
                error=str(e),
                error_type=type(e).__name__
            )
            await db.rollback()
            raise
```

### TaskIQ 任务日志

```python
from core.broker import broker
from loguru import logger

@broker.task
async def task_process_data(data_id: str) -> dict:
    """异步任务 - 带完整日志"""
    
    logger.info(
        "Task started",
        task_name="process_data",
        data_id=data_id
    )
    
    try:
        # 执行任务逻辑
        result = await process_data_logic(data_id)
        
        logger.info(
            "Task completed",
            task_name="process_data",
            data_id=data_id,
            result_size=len(result)
        )
        
        return result
        
    except Exception as e:
        logger.error(
            "Task failed",
            task_name="process_data",
            data_id=data_id,
            error=str(e),
            error_type=type(e).__name__
        )
        raise
```

## 日志级别使用规范

### 级别定义
- **DEBUG**: 详细的调试信息，仅开发环境使用
- **INFO**: 一般信息，记录正常的业务流程
- **WARNING**: 警告信息，不影响功能但需要注意
- **ERROR**: 错误信息，功能执行失败
- **CRITICAL**: 严重错误，系统级别问题

### 使用场景

```python
# DEBUG - 调试信息
logger.debug("Query parameters", params=query_params)
logger.debug("Database query", sql=query_string)

# INFO - 业务流程
logger.info("User logged in", user_id=user_id)
logger.info("Order created", order_id=order_id, amount=amount)

# WARNING - 需要注意的情况
logger.warning("API rate limit approaching", user_id=user_id, requests=count)
logger.warning("Cache miss", key=cache_key)

# ERROR - 业务错误
logger.error("Payment failed", order_id=order_id, error=str(e))
logger.error("Email sending failed", user_id=user_id, error=str(e))

# CRITICAL - 系统级错误
logger.critical("Database connection lost", error=str(e))
logger.critical("NATS broker unavailable", error=str(e))
```

## 日志最佳实践

### 1. 使用结构化日志
```python
# ✅ 推荐：结构化日志
logger.info("User action", user_id=user_id, action="login", ip=ip_address)

# ❌ 避免：字符串拼接
logger.info(f"User {user_id} performed action login from {ip_address}")
```

### 2. 包含上下文信息
```python
# 包含足够的上下文信息便于排查问题
logger.error(
    "Order processing failed",
    order_id=order_id,
    user_id=user_id,
    product_id=product_id,
    amount=amount,
    error=str(e),
    error_type=type(e).__name__,
    traceback=traceback.format_exc()
)
```

### 3. 避免敏感信息
```python
# ✅ 推荐：脱敏处理
logger.info("User login", email=mask_email(user.email))

# ❌ 避免：记录敏感信息
logger.info("User login", password=password)  # 不要记录密码
logger.info("Payment info", card_number=card_number)  # 不要记录卡号
```

### 4. 合理的日志频率
```python
# ✅ 推荐：关键节点记录
logger.info("Batch processing started", batch_size=1000)
# ... 处理1000条数据
logger.info("Batch processing completed", processed=1000, failed=5)

# ❌ 避免：循环中大量日志
for item in items:
    logger.info("Processing item", item_id=item.id)  # 避免在循环中记录
```

### 5. 异常日志包含堆栈
```python
try:
    result = await process_data()
except Exception as e:
    logger.exception("Data processing failed")  # 自动包含堆栈信息
    # 或者
    logger.error("Data processing failed", exc_info=True)
```
