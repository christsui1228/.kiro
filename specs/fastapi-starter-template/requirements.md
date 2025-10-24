# FastAPI Starter Template 需求文档

## 简介

从 OneManage 项目中提取通用基础设施，创建一个可复用的 FastAPI 项目模板。该模板包含数据库、消息队列、对象存储等常用组件的配置和工具，让新项目可以快速启动。

## 术语表

- **模板项目 (Template Project)**: 位于 `~/coding/fastapi-starter-template/` 的独立项目
- **源项目 (Source Project)**: OneManage 项目，作为提取源
- **目标项目 (Target Project)**: 使用模板创建的新项目
- **初始化脚本 (Init Script)**: 用于配置新项目参数的自动化脚本
- **验证脚本 (Validation Script)**: 用于测试配置是否正确的工具脚本
- **配置参数化 (Config Parameterization)**: 将硬编码值替换为可配置的占位符

---

## 需求 1: 核心配置管理

**用户故事**: 作为开发者，我希望模板包含完整的配置管理系统，这样我可以通过环境变量快速配置新项目。

### 验收标准

1.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `core/config.py` 文件，该文件使用 Pydantic Settings 管理配置

1.2 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `.env.example` 文件，该文件包含所有必需的配置项占位符

1.3 WHEN 开发者设置环境变量 ENV 为 "development" 时，THE 配置系统 SHALL 自动加载 `.env.development` 文件

1.4 WHEN 开发者设置环境变量 ENV 为 "production" 时，THE 配置系统 SHALL 自动加载 `.env.production` 文件

1.5 WHEN 配置文件缺少必需字段时，THE 配置系统 SHALL 抛出清晰的错误信息，指明缺少哪些字段

---

## 需求 2: 数据库连接管理

**用户故事**: 作为开发者，我希望模板包含数据库连接管理，这样我可以直接使用异步数据库操作。

### 验收标准

2.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `core/database.py` 文件，该文件提供异步数据库会话管理

2.2 WHEN 模板项目被创建时，THE 数据库配置 SHALL 支持通过独立环境变量配置 DB_USER、DB_PASSWORD、DB_HOST、DB_PORT、DB_NAME

2.3 WHEN 应用启动时，THE 数据库连接 SHALL 使用连接池管理连接

2.4 WHEN 数据库操作完成时，THE 数据库会话 SHALL 自动关闭并返回连接池

---

## 需求 3: 消息队列配置

**用户故事**: 作为开发者，我希望模板包含 TaskIQ + NATS 消息队列配置，这样我可以快速实现异步任务处理。

### 验收标准

3.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `core/broker.py` 文件，该文件配置 TaskIQ 和 NATS JetStream

3.2 WHEN 模板项目被创建时，THE 消息队列配置 SHALL 支持通过 NATS_URL 环境变量配置连接地址

3.3 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `events/` 目录，该目录提供事件发布和订阅的基础设施

3.4 WHEN 开发者定义新的异步任务时，THE 任务 SHALL 能够通过 `@broker.task` 装饰器注册

---

## 需求 4: 对象存储客户端

**用户故事**: 作为开发者，我希望模板包含 S3 兼容的对象存储客户端，这样我可以快速实现文件上传下载功能。

### 验收标准

4.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `common/oss.py` 文件，该文件提供 S3 兼容的存储客户端

4.2 WHEN 模板项目被创建时，THE OSS 配置 SHALL 支持阿里云 OSS 和 Cloudflare R2

4.3 WHEN 开发者调用上传方法时，THE OSS 客户端 SHALL 支持异步上传文件

4.4 WHEN 开发者调用下载方法时，THE OSS 客户端 SHALL 支持异步下载文件

4.5 WHEN 开发者调用生成预签名 URL 方法时，THE OSS 客户端 SHALL 返回带过期时间的访问 URL

---

## 需求 5: 日志系统

**用户故事**: 作为开发者，我希望模板包含结构化日志系统，这样我可以方便地追踪应用运行状态。

### 验收标准

5.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `common/logging_config.py` 文件，该文件配置 Loguru 日志系统

5.2 WHEN 应用在开发环境运行时，THE 日志系统 SHALL 输出彩色格式化日志到控制台

5.3 WHEN 应用在生产环境运行时，THE 日志系统 SHALL 输出 JSON 格式日志

5.4 WHEN 日志记录时，THE 日志 SHALL 包含时间戳、日志级别、模块名称、消息内容

---

## 需求 6: 统一响应格式

**用户故事**: 作为开发者，我希望模板包含统一的 API 响应格式，这样前端可以标准化处理响应。

### 验收标准

6.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `common/response_envelope.py` 文件，该文件定义统一响应格式

6.2 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `common/envelope_middleware.py` 文件，该文件自动包装 API 响应

6.3 WHEN API 返回成功响应时，THE 响应格式 SHALL 包含 code、message、data 字段

6.4 WHEN API 返回错误响应时，THE 响应格式 SHALL 包含 code、message、error 字段

---

## 需求 7: 配置验证脚本

**用户故事**: 作为开发者，我希望模板包含配置验证脚本，这样我可以快速检查环境配置是否正确。

### 验收标准

7.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `scripts/test_db_connection.py` 脚本，该脚本测试数据库连接

7.2 WHEN 开发者运行数据库测试脚本时，THE 脚本 SHALL 尝试连接数据库并执行简单查询

7.3 WHEN 数据库连接成功时，THE 脚本 SHALL 输出成功消息和数据库版本信息

7.4 WHEN 数据库连接失败时，THE 脚本 SHALL 输出详细的错误信息和配置检查建议

7.5 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `scripts/test_oss_connection.py` 脚本，该脚本测试 OSS 连接

7.6 WHEN 开发者运行 OSS 测试脚本时，THE 脚本 SHALL 尝试列举存储桶并上传测试文件

7.7 WHEN OSS 连接成功时，THE 脚本 SHALL 输出成功消息和存储桶信息

7.8 WHEN OSS 连接失败时，THE 脚本 SHALL 输出详细的错误信息和配置检查建议

7.9 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `scripts/test_nats_connection.py` 脚本，该脚本测试 NATS 连接

7.10 WHEN 开发者运行 NATS 测试脚本时，THE 脚本 SHALL 尝试连接 NATS 服务器并发布测试消息

7.11 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `scripts/health_check.py` 脚本，该脚本一键检查所有服务

7.12 WHEN 开发者运行健康检查脚本时，THE 脚本 SHALL 依次检查数据库、NATS、OSS 的连接状态

---

## 需求 8: 项目初始化脚本

**用户故事**: 作为开发者，我希望模板包含初始化脚本，这样我可以快速配置新项目的参数。

### 验收标准

8.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `scripts/init_project.py` 脚本

8.2 WHEN 开发者运行初始化脚本时，THE 脚本 SHALL 接受项目名称、数据库名称、OSS 存储桶等参数

8.3 WHEN 初始化脚本执行时，THE 脚本 SHALL 复制 `.env.example` 到 `.env.development`

8.4 WHEN 初始化脚本执行时，THE 脚本 SHALL 替换配置文件中的占位符为用户提供的参数

8.5 WHEN 初始化脚本执行时，THE 脚本 SHALL 生成随机的 SECRET_KEY

8.6 WHEN 初始化脚本执行时，THE 脚本 SHALL 更新 `pyproject.toml` 中的项目名称和描述

8.7 WHEN 初始化脚本完成时，THE 脚本 SHALL 输出下一步操作指引

---

## 需求 9: 数据库迁移配置

**用户故事**: 作为开发者，我希望模板包含数据库迁移配置，这样我可以方便地管理数据库 schema 变更。

### 验收标准

9.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `alembic.ini` 配置文件

9.2 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `migrations/env.py` 文件，该文件配置 Alembic 环境

9.3 WHEN 开发者运行 `alembic revision` 命令时，THE 系统 SHALL 生成新的迁移脚本

9.4 WHEN 开发者运行 `alembic upgrade head` 命令时，THE 系统 SHALL 应用所有待执行的迁移

---

## 需求 10: Docker 开发环境

**用户故事**: 作为开发者，我希望模板包含 Docker 配置，这样我可以快速启动本地开发环境。

### 验收标准

10.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `docker-compose.yml` 文件

10.2 WHEN 开发者运行 `docker-compose up` 命令时，THE 系统 SHALL 启动 PostgreSQL 数据库服务

10.3 WHEN 开发者运行 `docker-compose up` 命令时，THE 系统 SHALL 启动 NATS JetStream 服务

10.4 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `Dockerfile` 文件，该文件用于构建应用镜像

---

## 需求 11: 示例业务模块

**用户故事**: 作为开发者，我希望模板包含示例业务模块，这样我可以参考标准的代码结构。

### 验收标准

11.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `features/example/` 目录

11.2 WHEN 模板项目被创建时，THE 示例模块 SHALL 包含 `models.py` 文件，展示 SQLModel 模型定义

11.3 WHEN 模板项目被创建时，THE 示例模块 SHALL 包含 `schemas.py` 文件，展示 Pydantic Schema 定义

11.4 WHEN 模板项目被创建时，THE 示例模块 SHALL 包含 `routers.py` 文件，展示 FastAPI 路由定义

11.5 WHEN 模板项目被创建时，THE 示例模块 SHALL 包含 `services.py` 文件，展示业务逻辑实现

11.6 WHEN 模板项目被创建时，THE 示例模块 SHALL 包含 `tasks.py` 文件，展示 TaskIQ 异步任务定义

---

## 需求 12: 使用文档

**用户故事**: 作为开发者，我希望模板包含详细的使用文档，这样我可以快速上手。

### 验收标准

12.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `README.md` 文件

12.2 WHEN 开发者阅读 README 时，THE 文档 SHALL 包含快速开始指南

12.3 WHEN 开发者阅读 README 时，THE 文档 SHALL 包含所有配置项的说明

12.4 WHEN 开发者阅读 README 时，THE 文档 SHALL 包含常用命令列表

12.5 WHEN 开发者阅读 README 时，THE 文档 SHALL 包含项目结构说明

12.6 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `docs/` 目录，该目录包含详细的技术文档

---

## 需求 13: 依赖管理

**用户故事**: 作为开发者，我希望模板使用 PDM 管理依赖，这样我可以保持与 OneManage 一致的开发体验。

### 验收标准

13.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `pyproject.toml` 文件

13.2 WHEN 模板项目被创建时，THE 依赖配置 SHALL 包含 FastAPI、Uvicorn、SQLModel、Alembic、TaskIQ、Loguru 等核心依赖

13.3 WHEN 开发者运行 `pdm install` 命令时，THE 系统 SHALL 安装所有必需的依赖

13.4 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `pdm.lock` 文件，锁定依赖版本

---

## 需求 14: 代码质量工具

**用户故事**: 作为开发者，我希望模板包含代码质量工具配置，这样我可以保持代码风格一致。

### 验收标准

14.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `.gitignore` 文件

14.2 WHEN 模板项目被创建时，THE pyproject.toml SHALL 包含 Black 代码格式化配置

14.3 WHEN 模板项目被创建时，THE pyproject.toml SHALL 包含 Ruff 代码检查配置

14.4 WHEN 开发者运行 `pdm run black .` 命令时，THE 系统 SHALL 格式化所有 Python 文件

---

## 需求 15: Makefile 快捷命令

**用户故事**: 作为开发者，我希望模板包含 Makefile，这样我可以使用简短的命令执行常用操作。

### 验收标准

15.1 WHEN 模板项目被创建时，THE 模板项目 SHALL 包含 `Makefile` 文件

15.2 WHEN 开发者运行 `make init` 命令时，THE 系统 SHALL 执行项目初始化流程

15.3 WHEN 开发者运行 `make test` 命令时，THE 系统 SHALL 执行所有配置验证脚本

15.4 WHEN 开发者运行 `make dev` 命令时，THE 系统 SHALL 启动开发环境

15.5 WHEN 开发者运行 `make health` 命令时，THE 系统 SHALL 执行健康检查

15.6 WHEN 开发者运行 `make format` 命令时，THE 系统 SHALL 格式化代码

15.7 WHEN 开发者运行 `make migrate` 命令时，THE 系统 SHALL 执行数据库迁移
