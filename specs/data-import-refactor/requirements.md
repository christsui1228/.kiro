# 数据导入模块重构需求文档

## 简介

本文档定义了 `features/data_import` 模块的重构需求。当前模块存在代码重复、职责不清、违背 SQLModel 设计原则等问题。重构目标是建立清晰的分层架构，提升代码可维护性和可读性。

## 术语表

- **SQLModel**: 结合 Pydantic 和 SQLAlchemy 的 ORM 框架，支持单一模型定义
- **瘦路由**: Router 层只负责路由定义、参数验证和依赖注入，不包含业务逻辑
- **Processor**: 业务逻辑处理层，负责核心业务流程编排
- **CRUD**: 数据库访问层，提供基础的增删改查操作
- **TaskIQ**: 异步任务队列框架，用于处理长时间运行的任务
- **NATS**: 消息中间件，支持事件驱动架构
- **Polars**: 高性能数据处理库，用于 DataFrame 操作
- **AsyncSession**: SQLAlchemy 异步数据库会话

## 需求

### 需求 1: 合并 models.py 和 schemas.py

**用户故事**: 作为开发者，我希望使用 SQLModel 的单一模型定义特性，避免在 models.py 和 schemas.py 中重复定义相同的字段，以便减少维护成本和出错概率。

#### 验收标准

1. WHEN 定义数据库表模型时，THE 系统 SHALL 在 models.py 中使用 SQLModel 的 `table=True` 参数创建表模型
2. WHEN 定义 API 请求模型时，THE 系统 SHALL 在 models.py 中创建继承自表模型的 Create 类，不设置 `table=True` 参数
3. WHEN 定义 API 更新模型时，THE 系统 SHALL 在 models.py 中创建 Update 类，所有字段使用现代联合类型语法（`字段类型 | None = None`）
4. WHEN 定义 API 响应模型时，THE 系统 SHALL 在 models.py 中创建 Read 类，包含数据库生成的字段（id, created_at, updated_at）
5. THE 系统 SHALL 删除 schemas.py 文件，所有模型定义统一在 models.py 中

### 需求 2: 重构 CRUD 层为标准格式

**用户故事**: 作为开发者，我希望 CRUD 层遵循 slim-template 的标准格式，提供简洁、一致的数据库访问接口，以便提高代码可读性和团队协作效率。

#### 验收标准

1. THE 系统 SHALL 将 crud.py 重命名为 crud_async.py
2. WHEN 定义 CRUD 函数时，THE 系统 SHALL 使用简洁的函数签名，参数为 `db: AsyncSession` 和业务参数
3. WHEN 创建记录时，THE 系统 SHALL 使用 `Model.model_validate(payload)` 进行模型转换
4. WHEN 更新记录时，THE 系统 SHALL 使用 `payload.model_dump(exclude_unset=True)` 获取更新字段
5. WHEN 查询不到记录时，THE 系统 SHALL 抛出 `ValueError` 异常，包含清晰的错误信息
6. THE 系统 SHALL 在每个 CRUD 函数中包含简洁的中文文档字符串
7. THE 系统 SHALL 保留 `bulk_upsert_orders` 函数的业务逻辑（completion_date 条件判断）

### 需求 3: 建立 Processor 业务逻辑层

**用户故事**: 作为开发者，我希望将所有业务逻辑集中在 processor.py 中，实现清晰的分层架构，以便业务逻辑可以被 router 和 tasks 复用。

#### 验收标准

1. WHEN 处理文件上传时，THE 系统 SHALL 在 processor.py 中提供 `process_file_upload` 函数，接收 UploadFile 和用户信息
2. WHEN 执行导入任务时，THE 系统 SHALL 在 processor.py 中提供 `process_import_task` 函数，接收 task_id 和 file_path
3. WHEN 解析文件时，THE 系统 SHALL 在 processor.py 中提供 `parse_file` 函数，支持 CSV 和 Excel 格式
4. WHEN 生成报告时，THE 系统 SHALL 在 processor.py 中提供 `generate_report` 函数，创建导入结果报告
5. WHEN 处理导入流程时，THE 系统 SHALL 按照以下顺序执行：更新状态为 PROCESSING → 解析文件 → 验证数据 → 保存数据 → 生成报告 → 更新最终状态 → 发布事件 → 清理临时文件
6. THE 系统 SHALL 在 processor.py 中统一处理异常，更新任务状态为 FAILED 并记录错误信息

### 需求 4: 实现瘦路由

**用户故事**: 作为开发者，我希望 router.py 只负责路由定义和参数验证，将业务逻辑委托给 processor 层，以便提高代码的可测试性和可维护性。

#### 验收标准

1. WHEN 定义路由函数时，THE 系统 SHALL 只包含依赖注入、参数验证和调用 processor 函数
2. WHEN 处理文件上传时，THE 系统 SHALL 调用 `processor.process_file_upload` 并启动异步任务
3. WHEN 查询任务状态时，THE 系统 SHALL 直接调用 CRUD 函数获取任务信息
4. WHEN 下载报告时，THE 系统 SHALL 验证报告存在性后返回 FileResponse
5. THE 系统 SHALL 从 router.py 中移除所有业务逻辑函数（save_upload_file, parse_file, import_task）
6. THE 系统 SHALL 保持路由函数的简洁性，每个函数不超过 15 行代码

### 需求 5: 简化 TaskIQ 任务

**用户故事**: 作为开发者，我希望 tasks.py 只负责任务定义和调用 processor，避免重复实现业务逻辑，以便保持代码的 DRY 原则。

#### 验收标准

1. WHEN 定义 TaskIQ 任务时，THE 系统 SHALL 使用 `@broker.task` 装饰器
2. WHEN 执行导入任务时，THE 系统 SHALL 调用 `processor.process_import_task` 函数
3. THE 系统 SHALL 从 tasks.py 中移除所有重复的业务逻辑代码
4. THE 系统 SHALL 在任务函数中只包含参数转换和 processor 调用
5. THE 系统 SHALL 保持任务函数的简洁性，不超过 10 行代码

### 需求 6: 保持向后兼容

**用户故事**: 作为系统维护者，我希望重构后的 API 接口保持向后兼容，不影响现有的客户端调用，以便平滑过渡到新架构。

#### 验收标准

1. THE 系统 SHALL 保持所有 API 端点的路径不变
2. THE 系统 SHALL 保持所有 API 端点的请求和响应格式不变
3. THE 系统 SHALL 保持数据库表结构不变
4. THE 系统 SHALL 保持 TaskIQ 任务的函数签名不变
5. THE 系统 SHALL 保持 NATS 事件的格式不变

### 需求 7: 代码质量和规范

**用户故事**: 作为开发者，我希望重构后的代码遵循项目规范和最佳实践，以便提高代码质量和团队协作效率。

#### 验收标准

1. THE 系统 SHALL 使用现代 Python 类型注解语法（`| None` 而非 `Optional`）
2. THE 系统 SHALL 为所有公共函数提供中文文档字符串
3. THE 系统 SHALL 使用 loguru 记录关键操作日志
4. THE 系统 SHALL 使用 AsyncSession 进行所有数据库操作
5. THE 系统 SHALL 遵循 Black 代码格式化规范
6. THE 系统 SHALL 在适当位置添加类型提示，提高代码可读性
