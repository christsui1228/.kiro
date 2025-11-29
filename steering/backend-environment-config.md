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

### 配置类设计（嵌套结构）

#### 1. 基础配置和枚举
```python
from pydantic import BaseModel, Field
from pydantic_settings import BaseSettings, SettingsConfigDict
from functools import lru_cache
from pathlib import Path
import os
from enum import Enum

ROOT_DIR = Path(__file__).resolve().parent.parent

class Environment(str, Enum):
    """环境类型枚举"""
    DEVELOPMENT = "development"
    TESTING = "testing"
    PRODUCTION = "production"
```

#### 2. 嵌套配置类
```python
class DatabaseConfig(BaseModel):
    """数据库配置"""
    user: str = Field(..., description="数据库用户名")
    password: str = Field(..., description="数据库密码")
    host: str = Field(default="localhost", description="数据库主机地址")
    port: int = Field(default=5432, description="数据库端口")
    name: str = Field(..., description="数据库名称")
    echo_log: bool = Field(default=False, description="是否打印 SQL 日志")
    
    # 连接池配置
    pool_size: int = Field(default=10, description="连接池大小")
    max_overflow: int = Field(default=20, description="连接池最大溢出")
    
    @property
    def async_url(self) -> str:
        """异步数据库连接 URL"""
        return f"postgresql+asyncpg://{self.user}:{self.password}@{self.host}:{self.port}/{self.name}"
    
    @property
    def sync_url(self) -> str:
        """同步数据库连接 URL"""
        return f"postgresql+psycopg://{self.user}:{self.password}@{self.host}:{self.port}/{self.name}"

class NATSConfig(BaseModel):
    """NATS 消息队列配置"""
    url: str = Field(
        default="nats://localhost:4222",
        description="NATS 服务器地址",
        validation_alias="NATS_SERVER_URL"  # 支持云效环境变量别名
    )
    stream: str = Field(default="taskiq", description="JetStream 流名称")
    user: str | None = Field(default=None, description="认证用户名")
    password: str | None = Field(default=None, description="认证密码")
    
    @property
    def servers(self) -> list[str]:
        """转换为 taskiq-nats 需要的服务器地址列表格式"""
        return [self.url]

class S3Config(BaseModel):
    """S3 对象存储配置"""
    endpoint: str | None = Field(default=None, description="S3 服务端点")
    access_key_id: str | None = Field(default=None, description="访问密钥 ID")
    secret_access_key: str | None = Field(default=None, description="访问密钥")
    bucket: str | None = Field(default=None, description="存储桶名称")
    region: str = Field(default="auto", description="区域")

class FileStorageConfig(BaseModel):
    """文件存储配置"""
    upload_dir: str = Field(default="/data/uploads", description="上传文件目录")
    temp_dir: str = Field(default="/tmp", description="临时文件目录")
    
    @property
    def temp_uploads_dir(self) -> Path:
        """PSD上传临时目录"""
        path = Path(self.temp_dir) / "psd_uploads"
        path.mkdir(parents=True, exist_ok=True)
        return path
```

#### 3. 主配置类
```python
class Settings(BaseSettings):
    """主配置类"""
    env: Environment = Field(default=Environment.DEVELOPMENT, description="运行环境")
    debug: bool = Field(default=False, description="调试模式")
    
    # 嵌套配置组
    database: DatabaseConfig
    nats: NATSConfig
    s3: S3Config
    file_storage: FileStorageConfig
    
    model_config = SettingsConfigDict(
        env_file=(
            ROOT_DIR / f".env.{os.getenv('ENV', 'development').lower()}",
            ROOT_DIR / ".env"
        ),
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="allow",
        env_nested_delimiter="__"  # 嵌套配置使用双下划线分隔
    )
```

#### 4. 配置加载逻辑
```python
@lru_cache
def get_settings() -> Settings:
    """获取配置实例（带缓存）"""
    try:
        settings_instance = Settings()
        
        # 只在开发环境打印详细信息
        if settings_instance.env == Environment.DEVELOPMENT:
            logger.info(f"✅ 配置加载成功: {settings_instance.env.value}")
            logger.info(f"🗄️  数据库: {settings_instance.database.host}")
            logger.info(f"📨 NATS: {settings_instance.nats.url}")
        
        return settings_instance
    except Exception as e:
        logger.error(f"❌ 配置加载失败: {e}")
        raise

settings = get_settings()
```

### 环境变量命名规范（嵌套配置）

使用双下划线 `__` 分隔嵌套层级：

```bash
# 顶层配置
ENV=development
DEBUG=true

# 数据库配置（DATABASE__ 前缀）
DATABASE__USER=postgres
DATABASE__PASSWORD=postgres
DATABASE__HOST=localhost
DATABASE__PORT=5432
DATABASE__NAME=onemanage_dev
DATABASE__ECHO_LOG=true

# NATS 配置（NATS__ 前缀）
NATS__URL=nats://localhost:4222
NATS__STREAM=taskiq_dev
NATS__USER=
NATS__PASSWORD=

# S3 配置（S3__ 前缀）
S3__ENDPOINT=https://oss-cn-hangzhou.aliyuncs.com
S3__ACCESS_KEY_ID=your_key
S3__SECRET_ACCESS_KEY=your_secret
S3__BUCKET=your_bucket

# 文件存储配置（FILE_STORAGE__ 前缀）
FILE_STORAGE__UPLOAD_DIR=/data/uploads
FILE_STORAGE__TEMP_DIR=/tmp
```

## 开发环境配置

### .env.development 示例
```bash
# ==================== 核心配置 ====================
# 运行环境: development, testing, production
ENV=development

# 调试模式
DEBUG=true


# ==================== 数据库配置 ====================
# PostgreSQL 数据库用户名
DATABASE__USER=postgres

# PostgreSQL 数据库密码
DATABASE__PASSWORD=postgres

# PostgreSQL 数据库主机
DATABASE__HOST=localhost

# PostgreSQL 数据库端口
DATABASE__PORT=5432

# PostgreSQL 数据库名称
DATABASE__NAME=onemanage_dev

# 是否打印 SQL 日志（开发环境建议 true，生产环境建议 false）
DATABASE__ECHO_LOG=true

# 连接池配置（可选，有默认值）
DATABASE__POOL_SIZE=10
DATABASE__MAX_OVERFLOW=20
DATABASE__POOL_RECYCLE=180


# ==================== NATS 消息队列配置 ====================
# NATS 服务器地址
NATS__URL=nats://localhost:4222

# JetStream 流名称
NATS__STREAM=taskiq_dev

# NATS 认证用户名（可选）
# NATS__USER=admin

# NATS 认证密码（可选）
# NATS__PASSWORD=your-nats-password


# ==================== S3 对象存储配置（可选） ====================
# 如果不需要对象存储，可以注释掉这部分
# 支持阿里云 OSS、Cloudflare R2、AWS S3、MinIO 等

# S3 服务端点
# S3__ENDPOINT=https://oss-cn-hangzhou.aliyuncs.com

# S3 访问密钥 ID
# S3__ACCESS_KEY_ID=your_access_key

# S3 访问密钥
# S3__SECRET_ACCESS_KEY=your_secret_key

# S3 存储桶名称
# S3__BUCKET=your_bucket

# S3 对象键前缀（可选）
# S3__PREFIX=dev

# S3 区域
# S3__REGION=auto


# ==================== 文件存储配置 ====================
# 上传文件目录
FILE_STORAGE__UPLOAD_DIR=/data/uploads

# 报告文件目录
FILE_STORAGE__REPORT_DIR=/data/reports

# 临时文件目录
FILE_STORAGE__TEMP_DIR=/tmp


# ==================== 使用说明 ====================
# 1. 复制此文件为 .env.development
# 2. 根据实际情况修改配置值
# 3. 不需要的配置可以注释掉或删除
# 4. 敏感信息（密码、密钥）不要提交到版本控制
```

### .env.example 模板
```bash
# ==================== 核心配置 ====================
# 运行环境: development, testing, production
ENV=development

# 调试模式
DEBUG=false


# ==================== 数据库配置 ====================
# PostgreSQL 数据库用户名
DATABASE__USER=your_db_user

# PostgreSQL 数据库密码
DATABASE__PASSWORD=your_db_password

# PostgreSQL 数据库主机
DATABASE__HOST=localhost

# PostgreSQL 数据库端口
DATABASE__PORT=5432

# PostgreSQL 数据库名称
DATABASE__NAME=your_db_name

# 是否打印 SQL 日志（开发环境建议 true，生产环境建议 false）
DATABASE__ECHO_LOG=false

# 连接池配置（可选，有默认值）
DATABASE__POOL_SIZE=10
DATABASE__MAX_OVERFLOW=20
DATABASE__POOL_RECYCLE=180


# ==================== NATS 消息队列配置 ====================
# NATS 服务器地址
NATS__URL=nats://localhost:4222

# JetStream 流名称
NATS__STREAM=taskiq

# NATS 认证用户名（可选）
# NATS__USER=

# NATS 认证密码（可选）
# NATS__PASSWORD=


# ==================== S3 对象存储配置（可选） ====================
# 如果不需要对象存储，可以注释掉这部分

# S3 服务端点
# 阿里云 OSS: https://oss-cn-hangzhou.aliyuncs.com
# Cloudflare R2: https://<account-id>.r2.cloudflarestorage.com
# AWS S3: https://s3.us-east-1.amazonaws.com
S3__ENDPOINT=https://oss-cn-hangzhou.aliyuncs.com

# S3 访问密钥 ID
S3__ACCESS_KEY_ID=your_access_key

# S3 访问密钥
S3__SECRET_ACCESS_KEY=your_secret_key

# S3 存储桶名称
S3__BUCKET=your_bucket_name

# S3 对象键前缀（可选）
S3__PREFIX=

# S3 区域
S3__REGION=cn-hangzhou


# ==================== 文件存储配置 ====================
# 上传文件目录
FILE_STORAGE__UPLOAD_DIR=/data/uploads

# 报告文件目录
FILE_STORAGE__REPORT_DIR=/data/reports

# 临时文件目录
FILE_STORAGE__TEMP_DIR=/tmp


# ==================== 使用说明 ====================
# 1. 复制此文件为 .env.development 或 .env.production
# 2. 根据实际情况修改配置值
# 3. 不需要的配置可以注释掉或删除
# 4. 敏感信息（密码、密钥）不要提交到版本控制
# 5. 生产环境必须使用强随机值作为密钥
```

## 云效部署配置

### 环境变量注入策略
在阿里云效中，通过构建环境变量注入生产配置：

```yaml
# 云效 Pipeline 环境变量配置
environment:
  ENV: production
  DEBUG: "false"
  
  # 数据库配置（使用 DATABASE__ 前缀）
  DATABASE__USER: ${DB_USER}
  DATABASE__PASSWORD: ${DB_PASSWORD}
  DATABASE__HOST: ${DB_HOST}
  DATABASE__PORT: "5432"
  DATABASE__NAME: ${DB_NAME}
  DATABASE__ECHO_LOG: "false"
  
  # NATS 配置（使用 NATS__ 前缀，支持别名）
  NATS_SERVER_URL: ${NATS_URL}  # 使用别名映射到 NATS__URL
  NATS__STREAM: ${NATS_STREAM}
  
  # S3 配置（使用 S3__ 前缀）
  S3__ENDPOINT: ${OSS_ENDPOINT}
  S3__ACCESS_KEY_ID: ${OSS_ACCESS_KEY}
  S3__SECRET_ACCESS_KEY: ${OSS_SECRET_KEY}
  S3__BUCKET: ${OSS_BUCKET}
  S3__REGION: ${OSS_REGION}
  
  # 文件存储配置（使用 FILE_STORAGE__ 前缀）
  FILE_STORAGE__UPLOAD_DIR: /data/uploads
  FILE_STORAGE__TEMP_DIR: /tmp
```

### Docker 环境变量传递
```dockerfile
# Dockerfile 中支持环境变量
ENV ENV=production
ENV DEBUG=false
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
      - DEBUG=false
      # 数据库配置
      - DATABASE__USER=${DB_USER}
      - DATABASE__PASSWORD=${DB_PASSWORD}
      - DATABASE__HOST=postgres
      - DATABASE__PORT=5432
      - DATABASE__NAME=${DB_NAME}
      # NATS 配置
      - NATS__URL=nats://nats:4222
      - NATS__STREAM=taskiq
      # 文件存储配置
      - FILE_STORAGE__TEMP_DIR=/tmp
    env_file:
      - .env.production  # 可选的环境文件
```

## 配置验证和调试

### 开发环境配置检查
```python
# 仅在开发环境显示敏感信息
if settings.env == Environment.DEVELOPMENT:
    logger.info(f"🚀 运行环境: {settings.env.value}")
    logger.info(f"🐛 调试模式: {'开启' if settings.debug else '关闭'}")
    logger.info(f"🗄️  数据库: {settings.database.host}:{settings.database.port}/{settings.database.name}")
    logger.info(f"📨 NATS: {settings.nats.url} (stream: {settings.nats.stream})")
    logger.info(f"📁 临时目录: {settings.file_storage.temp_dir}")
```

### 配置访问方式
```python
# 访问嵌套配置
database_url = settings.database.async_url
nats_servers = settings.nats.servers
s3_endpoint = settings.s3.endpoint
temp_dir = settings.file_storage.temp_uploads_dir

# 使用配置属性
async with create_async_engine(settings.database.async_url) as engine:
    # ...
```

### 配置文件存在性检查
```python
def validate_config_files():
    """验证配置文件是否存在"""
    env = os.getenv("ENV", "development").lower()
    specific_env_file = ROOT_DIR / f".env.{env}"
    default_env_file = ROOT_DIR / ".env"
    
    if not specific_env_file.exists() and not default_env_file.exists():
        logger.warning("⚠️  未找到 .env 文件，仅使用环境变量")
    else:
        config_source = specific_env_file if specific_env_file.exists() else default_env_file
        logger.info(f"✅ 配置加载自: {config_source}")
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
# 使用 Pydantic 类型验证和嵌套配置
class DatabaseConfig(BaseModel):
    port: int = 5432  # 自动类型转换
    echo_log: bool = False  # 布尔值验证
    pool_size: int = Field(default=10, description="连接池大小")
    
    # 可选字段使用 | None 类型
    user: str | None = None
    password: str | None = None

class NATSConfig(BaseModel):
    url: str = Field(default="nats://localhost:4222")
    stream: str = Field(default="taskiq")
    user: str | None = None  # 可选字段
    password: str | None = None  # 可选字段

class Settings(BaseSettings):
    env: Environment = Environment.DEVELOPMENT
    debug: bool = False
    
    # 嵌套配置（必填）
    database: DatabaseConfig
    nats: NATSConfig
    
    # 嵌套配置（可选）
    s3: S3Config | None = None
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
if settings.env == Environment.DEVELOPMENT:
    # 开发环境特定配置
    app.add_middleware(CORSMiddleware, allow_origins=["*"])
    
    # 开启 SQL 日志
    engine = create_async_engine(
        settings.database.async_url,
        echo=settings.database.echo_log
    )
    
elif settings.env == Environment.PRODUCTION:
    # 生产环境特定配置
    app.add_middleware(CORSMiddleware, allow_origins=["https://yourdomain.com"])
    
    # 关闭 SQL 日志
    engine = create_async_engine(
        settings.database.async_url,
        echo=False,
        pool_size=settings.database.pool_size,
        max_overflow=settings.database.max_overflow
    )
```

### 配置组合使用示例
```python
# 数据库连接
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(
    settings.database.async_url,
    echo=settings.database.echo_log,
    pool_size=settings.database.pool_size,
    max_overflow=settings.database.max_overflow
)

# NATS 连接
from taskiq_nats import NatsBroker

broker = NatsBroker(
    servers=settings.nats.servers,
    stream_name=settings.nats.stream,
    queue="workers"
)

# S3 客户端
import boto3

if settings.s3.endpoint:
    s3_client = boto3.client(
        's3',
        endpoint_url=settings.s3.endpoint,
        aws_access_key_id=settings.s3.access_key_id,
        aws_secret_access_key=settings.s3.secret_access_key,
        region_name=settings.s3.region
    )
```