---
inclusion: fileMatch
fileMatchPattern: 'docker-compose*.yml|Dockerfile|*.sh'
---

# 部署和基础设施规范

## 容器化部署标准

### Docker 规范
- **基础镜像**: `python:3.12.11-slim-bookworm`
- **镜像仓库**: 阿里云容器镜像服务 (ACR)
- **网络模式**: Docker 内部网络通信
- **数据持久化**: Docker Volume 挂载

### Docker Compose 架构
```yaml
# 标准服务组合
services:
  app:          # FastAPI 应用
  postgres:     # PostgreSQL 17 数据库
  nats:         # NATS 2.12 + JetStream
  traefik:      # Traefik 3.4 反向代理
```

### 环境配置管理
- **配置文件**: `.env` 文件管理环境变量
- **敏感信息**: Docker Secrets 或环境变量注入
- **多环境**: `docker-compose.dev.yml`, `docker-compose.prod.yml`

## CI/CD 流程规范

### 代码仓库管理
- **主分支**: `main` 分支作为生产部署源
- **分支策略**: Feature Branch → Pull Request → Main
- **提交规范**: 使用 Conventional Commits 格式

### 阿里云效集成
```yaml
# 构建流程
1. 代码推送到 main 分支
2. 触发阿里云效 Pipeline
3. 执行 Docker 镜像构建
4. 推送到 ACR 镜像仓库
5. 自动部署到目标环境
```

### 镜像构建规范
- **构建上下文**: 项目根目录
- **多阶段构建**: 开发依赖与生产依赖分离
- **镜像标签**: 使用 Git commit SHA 作为标签
- **镜像优化**: 使用 .dockerignore 减少构建上下文

## 反向代理配置

### Traefik 3.4 配置
- **服务发现**: Docker Provider 自动发现
- **SSL 终止**: Let's Encrypt 自动证书
- **负载均衡**: 轮询算法
- **健康检查**: HTTP 健康检查端点

### 路由规则
```yaml
# 标准路由配置
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.app.rule=Host(`api.domain.com`)"
  - "traefik.http.routers.app.tls.certresolver=letsencrypt"
  - "traefik.http.services.app.loadbalancer.server.port=8000"
```

## 云服务集成

### 阿里云 ECI 弹性计算
- **触发方式**: NATS 事件驱动
- **资源规格**: 按计算需求动态分配
- **网络连接**: VPC 内网通信
- **数据传输**: OSS 对象存储

### 阿里云 OSS 对象存储
- **SDK**: boto3 兼容接口
- **存储类型**: 标准存储 + 低频访问存储
- **访问控制**: RAM 角色授权
- **CDN 加速**: 可选配置阿里云 CDN

## 网络和安全

### 网络架构
```
Internet → Traefik (80/443) → FastAPI (8000)
                            → PostgreSQL (5432)
                            → NATS (4222)
```

### 安全配置
- **端口暴露**: 仅 Traefik 暴露 80/443 端口
- **内网通信**: 服务间使用 Docker 内部网络
- **数据库**: 不直接暴露到公网
- **API 安全**: HTTPS 强制重定向

## 监控和日志

### 基础监控
- **容器状态**: Docker Compose 健康检查
- **应用日志**: 标准输出 + loguru 结构化日志
- **错误追踪**: 基础异常日志记录

### 日志管理
```python
# 标准日志配置
from loguru import logger

logger.add(
    "logs/app.log",
    rotation="1 day",
    retention="30 days",
    format="{time} | {level} | {message}"
)
```

## 部署脚本规范

### Docker Compose 模板
```yaml
version: '3.8'

services:
  app:
    build: .
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/db
      - NATS_SERVERS=nats://nats:4222
    depends_on:
      - postgres
      - nats
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`api.domain.com`)"

  postgres:
    image: postgres:17
    environment:
      - POSTGRES_DB=onemanage
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  nats:
    image: nats:2.12
    command: ["-js", "-sd", "/data"]
    volumes:
      - nats_data:/data

  traefik:
    image: traefik:3.4
    command:
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

volumes:
  postgres_data:
  nats_data:
```

### 部署命令
```bash
# 生产环境部署
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 开发环境部署
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# 数据库迁移
docker-compose exec app pdm run alembic upgrade head

# 查看日志
docker-compose logs -f app
```

## 环境变量规范

### 必需环境变量
```bash
# 数据库配置
DATABASE_URL=postgresql://user:pass@postgres:5432/db
DB_PASSWORD=secure_password

# NATS 配置
NATS_SERVERS=nats://nats:4222
NATS_STREAM=onemanage
NATS_USER=optional_user
NATS_PASSWORD=optional_password

# 阿里云配置
OSS_ACCESS_KEY_ID=your_access_key
OSS_ACCESS_KEY_SECRET=your_secret_key
OSS_BUCKET_NAME=your_bucket
OSS_ENDPOINT=oss-cn-hangzhou.aliyuncs.com

# 应用配置
APP_ENV=production
SECRET_KEY=your_secret_key
DEBUG=false
```

## 故障排查

### 常见问题
1. **容器启动失败** - 检查环境变量和依赖服务
2. **数据库连接失败** - 验证 DATABASE_URL 和网络连通性
3. **NATS 连接失败** - 检查 NATS 服务状态和配置
4. **Traefik 路由失败** - 验证 labels 配置和域名解析

### 调试命令
```bash
# 查看容器状态
docker-compose ps

# 查看服务日志
docker-compose logs service_name

# 进入容器调试
docker-compose exec app bash

# 检查网络连通性
docker-compose exec app ping postgres
```

## 备份和恢复

### 数据备份
```bash
# PostgreSQL 数据备份
docker-compose exec postgres pg_dump -U postgres onemanage > backup.sql

# NATS JetStream 数据备份
docker-compose exec nats nats stream backup stream_name
```

### 灾难恢复
- **数据恢复**: 从 PostgreSQL 备份恢复
- **配置恢复**: 从 Git 仓库重新部署
- **服务重启**: Docker Compose 重新启动所有服务