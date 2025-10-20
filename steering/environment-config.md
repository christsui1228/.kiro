# 环境变量管理规范

## 配置管理策略

### 环境文件优先级
基于 OneManage 的配置设计，采用分层环境变量管理：

```
系统环境变量 > .env.{ENV} > .env > 默认值
```

### 环境文件命名规范
- **开发环境**: `.env.development` (默认)
- **测试环境**: `.env.testing`
- **生产环境**: `.env.production`
- **通用配置**: `.env` (兜底配置)
- **配置模板**: `.env.example` (版本控制)

### 环境变量控制
```bash
# 通过 ENV 环境变量控制加载哪个配置文件
ENV=development  # 加载 .env.development
ENV=production   # 加载 .env.production
ENV=testing      # 加载 .env.testing
```

## Pydantic Settings 配置规范

### 配置类设计
```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field
from typing import Union
from enum import Enum
import os
from pathlib import Path

class Environment(str, Enum):
    DEVELOPMENT = "development"
    TESTING = "testing"
    PRODUCTION = "production"

class Settings(BaseSettings):
    # 环境配置
    ENV: Environment = Environment.DEVELOPMENT
    
    # 数据库配置 (分散配置，便于云效注入)
    DB_USER: str
    DB_PASSWORD: str
    DB_HOST: str
    DB_PORT: str
    DB_NAME: str
    
    # 构建数据库连接URL的属性方法
    @property
    def database_url(self) -> str:
        return f"postgresql+asyncpg://{self.DB_USER}:{self.DB_PASSWORD}@{self.DB_HOST}:{self.DB_PORT}/{self.DB_NAME}"
    
    # NATS 配置 (支持云效环境变量别名)
    NATS_URL: str = Field(
        default="nats://localhost:4222",
        validation_alias="NATS_SERVER_URL"  # 云效可能使用不同的变量名，这种设计在本地可以自动切换到localhost:4222
    )
    
    # 配置文件加载规则
    model_config = SettingsConfigDict(
        env_file=(
            Path(__file__).parent.parent / f".env.{os.getenv('ENV', 'development').lower()}",
            Path(__file__).parent.parent / ".env"
        ),
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="allow"
    )
```

### 配置加载逻辑
```python
@lru_cache()
def get_settings() -> Settings:
    """获取配置实例，支持环境文件优先级"""
    env_priority = os.getenv("ENV", "development").lower()
    logger.info(f"Loading config for environment: {env_priority}")
    
    # 自动加载对应环境的配置文件
    return Settings()
```

## 开发环境配置

### .env.development 示例
```bash
# 环境标识
ENV=development

# 数据库配置
DB_USER=postgres
DB_PASSWORD=postgres
DB_HOST=localhost
DB_PORT=5432
DB_NAME=onemanage_dev

# NATS 配置
NATS_URL=nats://localhost:4222
NATS_STREAM=taskiq_dev
NATS_USER=
NATS_PASSWORD=

# 调试配置
DEBUG=true
DB_ECHO_LOG=true

# 本地存储配置
TEMP_DIR=/tmp
UPLOAD_DIR=/data/uploads
REPORT_DIR=/data/reports

# 阿里云配置 (开发环境可选)
S3_ENDPOINT=
S3_ACCESS_KEY_ID=
S3_SECRET_ACCESS_KEY=
S3_BUCKET=
```

### .env.example 模板
```bash
# .env.example（模板文件，纳入版本控制）

# 环境标识
ENV=development

# 数据库配置
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_HOST=localhost
DB_PORT=5432
DB_NAME=your_db_name

# JWT 配置
SECRET_KEY=your_secret_key_here
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=1440

# NATS 配置
NATS_URL=nats://localhost:4222
NATS_STREAM=taskiq
NATS_USER=optional_user
NATS_PASSWORD=optional_password

# 阿里云 OSS 配置
S3_ENDPOINT=https://oss-cn-hangzhou.aliyuncs.com
S3_ACCESS_KEY_ID=your_access_key
S3_SECRET_ACCESS_KEY=your_secret_key
S3_BUCKET=your_bucket_name
S3_REGION=cn-hangzhou

# 应用配置
DEBUG=false
DB_ECHO_LOG=false
TEMP_DIR=/tmp
```

## 云效部署配置

### 环境变量注入策略
在阿里云效中，通过构建环境变量注入生产配置：

```yaml
# 云效 Pipeline 环境变量配置
environment:
  ENV: production
  
  # 数据库配置
  DB_USER: ${DB_USER}
  DB_PASSWORD: ${DB_PASSWORD}
  DB_HOST: ${DB_HOST}
  DB_PORT: "5432"
  DB_NAME: ${DB_NAME}
  
  # NATS 配置
  NATS_SERVER_URL: ${NATS_URL}  # 使用别名映射
  NATS_STREAM: ${NATS_STREAM}
  
  # 阿里云配置
  S3_ENDPOINT: ${OSS_ENDPOINT}
  S3_ACCESS_KEY_ID: ${OSS_ACCESS_KEY}
  S3_SECRET_ACCESS_KEY: ${OSS_SECRET_KEY}
  S3_BUCKET: ${OSS_BUCKET}
```

### Docker 环境变量传递
```dockerfile
# Dockerfile 中支持环境变量
ENV ENV=production
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

# 运行时从环境变量获取配置
CMD ["pdm", "run", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker Compose 环境变量
```yaml
# docker-compose.prod.yml
services:
  app:
    environment:
      - ENV=production
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_HOST=postgres
      - NATS_URL=nats://nats:4222
    env_file:
      - .env.production  # 可选的环境文件
```

## 配置验证和调试

### 开发环境配置检查
```python
# 仅在开发环境显示敏感信息
if settings.ENV == Environment.DEVELOPMENT:
    logger.info(f"Environment: {settings.ENV.value}")
    logger.info(f"Database URL: {settings.database_url}")
    logger.info(f"NATS URL: {settings.NATS_URL}")
    logger.info(f"Debug mode: {settings.debug}")
```

### 配置文件存在性检查
```python
def validate_config_files():
    """验证配置文件是否存在"""
    env = os.getenv("ENV", "development").lower()
    specific_env_file = Path(f".env.{env}")
    default_env_file = Path(".env")
    
    if not specific_env_file.exists() and not default_env_file.exists():
        logger.warning("No .env files found, using environment variables only")
    else:
        logger.info(f"Config loaded from: {specific_env_file if specific_env_file.exists() else default_env_file}")
```

## 安全最佳实践

### 敏感信息处理
- ✅ **生产环境**: 使用云效环境变量注入
- ✅ **开发环境**: 使用 `.env.development` 文件
- ❌ **禁止**: 在代码中硬编码敏感信息
- ❌ **禁止**: 将 `.env.*` 文件提交到版本控制

### .gitignore 配置
```gitignore
# 环境配置文件
.env
.env.*
!.env.example  # 模板文件可以提交
```

### 配置字段类型安全
```python
# 使用 Pydantic 类型验证
class Settings(BaseSettings):
    DB_PORT: int = 5432  # 自动类型转换
    DEBUG: bool = False  # 布尔值验证
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 1440  # 整数验证
    
    # 可选字段使用 Union 类型
    NATS_USER: Union[str, None] = None
    NATS_PASSWORD: Union[str, None] = None
```

## 多环境管理

### 环境切换
```bash
# 本地开发
ENV=development pdm run uvicorn main:app --reload

# 测试环境
ENV=testing pdm run pytest

# 生产环境 (通过云效自动设置)
ENV=production
```

### 环境特定配置
```python
# 根据环境调整行为
if settings.ENV == Environment.DEVELOPMENT:
    # 开发环境特定配置
    app.add_middleware(CORSMiddleware, allow_origins=["*"])
elif settings.ENV == Environment.PRODUCTION:
    # 生产环境特定配置
    app.add_middleware(CORSMiddleware, allow_origins=["https://yourdomain.com"])
```