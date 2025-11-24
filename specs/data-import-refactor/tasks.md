# 数据导入模块重构任务列表

## 任务概述

本任务列表定义了 `features/data_import` 模块重构的具体实施步骤。采用一次性重构方案，按照依赖关系从底层到上层逐步实施。

## 实施任务

- [x] 1. 合并 models.py 和 schemas.py
  - 在 models.py 中使用 SQLModel 单一模型定义
  - 创建表模型（table=True）、Create 模型、Update 模型、Read 模型
  - 使用现代联合类型语法（`| None`）
  - 为所有模型添加中文文档字符串
  - _需求: 1.1, 1.2, 1.3, 1.4, 1.5, 7.1, 7.2_

- [x] 2. 重构 CRUD 层
  - [x] 2.1 创建 crud_async.py 文件
    - 按照 slim-template 格式编写 CRUD 函数
    - 实现 get_task、create_task、update_task 函数
    - 实现 get_order、get_orders 函数
    - 每个函数包含简洁的中文文档字符串
    - _需求: 2.1, 2.2, 2.6_
  
  - [x] 2.2 迁移 bulk_upsert_orders 函数
    - 从 crud.py 复制 bulk_upsert_orders 函数到 crud_async.py
    - 保留 completion_date 条件判断的业务逻辑
    - 保持函数签名和返回值不变
    - _需求: 2.7_
  
  - [x] 2.3 实现其他查询函数
    - 实现 get_tasks_by_user 函数
    - 实现 get_all_tasks 函数
    - 使用简洁的函数签名和类型注解
    - _需求: 2.2, 2.3, 2.4, 2.5_

- [x] 3. 重构 Processor 层
  - [x] 3.1 实现文件处理函数
    - 实现 save_upload_file 函数（保存上传文件）
    - 实现 parse_file 函数（解析 CSV/Excel）
    - 实现 cleanup_temp_file 函数（清理临时文件）
    - 添加详细的错误处理和日志记录
    - _需求: 3.1, 3.3, 3.6_
  
  - [x] 3.2 实现报告生成函数
    - 实现 generate_report 函数
    - 生成 CSV 格式的导入报告
    - 包含 order_id、status、message、version 字段
    - _需求: 3.4_
  
  - [x] 3.3 实现事件发布函数
    - 实现 publish_import_completed_event 函数
    - 发布到 NATS DATA_IMPORT_COMPLETED 主题
    - 包含 import_id、status、file_path、processed_items
    - _需求: 3.5_
  
  - [x] 3.4 实现文件上传处理函数
    - 实现 process_file_upload 函数
    - 验证文件格式（CSV/Excel）
    - 调用 crud.create_task 创建任务记录
    - 调用 save_upload_file 保存文件
    - 返回任务对象和文件路径
    - _需求: 3.1, 3.2_
  
  - [x] 3.5 实现核心导入流程函数
    - 实现 process_import_task 函数
    - 按照设计文档的流程顺序执行：
      1. 更新状态为 PROCESSING
      2. 解析文件
      3. 调用 service.validate_and_save_orders
      4. 生成报告
      5. 更新最终状态
      6. 发布事件
      7. 清理临时文件
    - 统一异常处理，更新任务状态为 FAILED
    - 使用 AsyncSessionFactory 创建数据库会话
    - 添加详细的日志记录
    - _需求: 3.2, 3.3, 3.4, 3.5, 3.6, 7.3, 7.4_

- [x] 4. 重构 Router 层
  - [x] 4.1 重构 upload_and_import 端点
    - 调用 processor.process_file_upload 处理文件上传
    - 启动异步任务 task_run_data_import.kiq
    - 提交数据库事务
    - 返回任务对象
    - 保持函数简洁（不超过 15 行）
    - _需求: 4.1, 4.2, 4.6_
  
  - [x] 4.2 重构 check_status 端点
    - 调用 crud.get_task 查询任务状态
    - 任务不存在时抛出 404 HTTPException
    - 返回任务对象
    - _需求: 4.1, 4.3_
  
  - [x] 4.3 重构 download_report 端点
    - 调用 crud.get_task 查询任务
    - 验证报告路径和文件存在性
    - 返回 FileResponse
    - _需求: 4.1, 4.4_
  
  - [x] 4.4 清理 router.py
    - 删除 save_upload_file 函数（已移到 processor）
    - 删除 parse_file 函数（已移到 processor）
    - 删除 import_task 函数（已移到 processor）
    - 更新导入语句，使用新的 models 和 crud_async
    - _需求: 4.5_

- [x] 5. 简化 Tasks 层
  - [x] 5.1 重构 task_run_data_import 任务
    - 保持 @broker.task 装饰器
    - 将 task_id 字符串转换为 UUID
    - 调用 processor.process_import_task
    - 添加简单的日志记录
    - 保持函数简洁（不超过 10 行）
    - _需求: 5.1, 5.2, 5.3, 5.4, 5.5_

- [x] 6. 更新导入语句和清理
  - [x] 6.1 更新所有模块的导入语句
    - router.py：从 models 导入模型，从 crud_async 导入 CRUD 函数
    - processor.py：从 models 导入模型，从 crud_async 导入 CRUD 函数
    - tasks.py：从 processor 导入处理函数
    - service.py：从 models 导入模型，从 crud_async 导入 CRUD 函数
    - _需求: 6.1, 6.2, 6.3, 6.4, 6.5_
  
  - [x] 6.2 删除废弃文件
    - 删除 schemas.py 文件
    - 删除 crud.py 文件（已被 crud_async.py 替代）
    - 考虑删除 raw_order_importer.py（如果不再使用）
    - _需求: 1.5_
  
  - [x] 6.3 更新 __init__.py
    - 更新模块导出，确保向后兼容
    - 导出新的 models 和 crud_async
    - _需求: 6.1, 6.2, 6.3, 6.4, 6.5_

- [x] 7. 代码质量检查
  - [x] 7.1 类型注解检查
    - 确保所有函数都有完整的类型注解
    - 使用现代联合类型语法（`| None`）
    - 运行 mypy 检查类型错误
    - _需求: 7.1, 7.6_
  
  - [x] 7.2 文档字符串检查
    - 确保所有公共函数都有中文文档字符串
    - 文档字符串包含参数说明、返回值说明、异常说明
    - _需求: 7.2_
  
  - [x] 7.3 代码格式化
    - 运行 Black 格式化所有修改的文件
    - 运行 isort 排序导入语句
    - _需求: 7.5_
  
  - [x] 7.4 日志记录检查
    - 确保关键操作都有日志记录
    - 使用合适的日志级别（INFO、WARNING、ERROR）
    - _需求: 7.3_

- [ ]* 8. 测试验证（可选）
  - [ ]* 8.1 手动测试 API 端点
    - 测试文件上传接口
    - 测试任务状态查询接口
    - 测试报告下载接口
    - _需求: 6.1, 6.2_
  
  - [ ]* 8.2 测试异步任务执行
    - 启动 TaskIQ Worker
    - 上传测试文件
    - 验证任务状态变化
    - 验证数据导入结果
    - 验证报告生成
    - _需求: 6.3, 6.4, 6.5_
  
  - [ ]* 8.3 测试事件发布
    - 启动事件消费者
    - 验证 DATA_IMPORT_COMPLETED 事件发布
    - 验证事件数据格式
    - _需求: 6.5_

## 任务依赖关系

```
1 (合并 models) 
  ↓
2 (重构 CRUD) 
  ↓
3 (重构 Processor) 
  ↓
4 (重构 Router) + 5 (简化 Tasks)
  ↓
6 (更新导入和清理)
  ↓
7 (代码质量检查)
  ↓
8 (测试验证) *可选
```

## 实施注意事项

### 向后兼容性
- 保持所有 API 端点路径不变
- 保持请求/响应格式不变
- 保持数据库表结构不变
- 保持 TaskIQ 任务函数签名不变
- 保持 NATS 事件格式不变

### 代码规范
- 使用现代 Python 类型注解语法（`| None`）
- 所有公共函数包含中文文档字符串
- 使用 loguru 记录关键操作日志
- 使用 AsyncSession 进行所有数据库操作
- 遵循 Black 代码格式化规范

### 错误处理
- Processor 层捕获所有异常，更新任务状态为 FAILED
- Router 层转换为 HTTPException
- CRUD 层抛出 ValueError（记录不存在）

### 性能考虑
- 使用 PostgreSQL INSERT ... ON CONFLICT 进行 upsert
- 分批处理（chunk_size=500）避免参数过多
- 及时清理临时文件
- 使用 Polars 流式处理大文件

## 验收标准

### 功能验收
- ✅ 文件上传接口正常工作
- ✅ 任务状态查询接口正常工作
- ✅ 报告下载接口正常工作
- ✅ 异步任务正常执行
- ✅ 数据导入逻辑正确（包含 upsert 规则）
- ✅ 报告正常生成
- ✅ 事件正常发布

### 代码质量验收
- ✅ 所有函数都有类型注解
- ✅ 所有公共函数都有中文文档字符串
- ✅ 代码通过 Black 格式化
- ✅ 代码通过 isort 排序
- ✅ 关键操作都有日志记录

### 架构验收
- ✅ Models 层：单一模型定义
- ✅ CRUD 层：遵循 slim-template 格式
- ✅ Processor 层：业务逻辑集中
- ✅ Router 层：瘦路由（每个函数不超过 15 行）
- ✅ Tasks 层：简洁（每个函数不超过 10 行）
- ✅ 职责清晰：service（数据验证）vs processor（流程编排）
