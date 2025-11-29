# ETL 模块重构任务列表

## 阶段 1：基础设施层

- [ ] 1.1 创建批处理服务
  - 创建文件 `OneManage/features/etl/services/batch.py`
  - 实现 `BatchProcessor` 类
  - 实现 `process_in_batches()` 静态方法，支持自定义批次大小和处理函数
  - 统一的错误处理和日志记录，返回 (成功数, 失败数) 元组
  - _需求: REQ-003 (批量处理优化)_

- [ ] 1.2 创建向量化查询工具
  - 创建文件 `OneManage/features/etl/services/vectorized_queries.py`
  - 实现 `VectorizedQueries` 类
  - 实现 `batch_lookup_customers()` 方法（批量查询客户）
  - 实现 `batch_lookup_leads()` 方法（批量查询线索）
  - 实现 `batch_lookup_order_type_mappings()` 方法（批量查询订单类型映射）
  - 使用 `core.database_sync.engine` 和 Polars `read_database_uri()` + ConnectorX
  - _需求: REQ-003, REQ-004 (向量化优化)_

## 阶段 2：订单 ETL 重构

- [ ] 2.1 重构订单 ETL 逻辑
  - 修改文件 `OneManage/features/etl/orders/etl.py`
  - 使用 `from core.database_sync import engine` 替代自定义引擎
  - 向量化字段映射：使用 `df.select([col.alias(), ...])` 替代循环
  - 使用 `BatchProcessor` 进行批量处理
  - 保持同步函数，返回 dict 结果
  - _需求: REQ-001, REQ-002, REQ-004, REQ-007_
  - _验收: 1000 条订单处理时间 < 5 秒_

- [ ] 2.2 重构订单 ETL 任务包装
  - 修改文件 `OneManage/features/etl/orders/tasks.py`
  - `run_etl_orders()` 使用 `asyncio.to_thread()` 包装同步 ETL
  - 事件监听器使用 `@event_task_handler` 装饰器
  - 统一错误处理和事件发布，使用 `events/subjects.py` 中的事件常量
  - _需求: REQ-001, REQ-005_

## 阶段 3：客户 ETL 重构

- [ ] 3.1 重构客户 ETL 逻辑
  - 修改文件 `OneManage/features/etl/customers/etl.py`
  - 使用 `core.database_sync.engine`
  - 向量化 join：使用 Polars `join()` 关联店铺配置
  - 使用 `anti` join 找出新客户（不在现有客户表中）
  - 向量化生成 UUID 和 customer_key
  - _需求: REQ-001, REQ-002, REQ-004, REQ-007_
  - _验收: 1000 条客户处理时间 < 3 秒_

- [ ] 3.2 重构客户 ETL 任务包装
  - 修改文件 `OneManage/features/etl/customers/tasks.py`
  - `run_etl_customers()` 使用 `asyncio.to_thread()` 包装
  - 事件监听器使用 `@event_task_handler` 装饰器
  - 监听 `ORDERS_ETL_COMPLETED` 事件
  - _需求: REQ-001, REQ-005_

## 阶段 4：商机 ETL 重构（重点优化）

- [ ] 4.1 重命名 leads ETL 文件
  - 重命名 `OneManage/features/etl/leads/etl_v2.py` 为 `etl.py`
  - 删除旧的 `etl.py` 文件（如果存在）
  - 更新 `tasks.py` 中的导入路径为 `from .etl import ...`
  - _需求: REQ-006_

- [ ] 4.2 重构商机 ETL 核心逻辑
  - 修改文件 `OneManage/features/etl/leads/etl.py`（重命名后）
  - 删除 `_get_engine()` 函数，使用 `core.database_sync.engine`
  - 使用 `VectorizedQueries` 批量查询客户、线索、订单类型映射
  - 向量化判断线索策略（new vs reuse）
  - 向量化聚合统计（sample_count, proof_count 等）
  - 向量化创建和更新线索
  - _需求: REQ-001, REQ-002, REQ-004, REQ-007_
  - _验收: 1000 条订单处理时间 < 10 秒（当前 15 秒，需提升 33%）_

- [ ] 4.3 重构商机 ETL 任务包装
  - 修改文件 `OneManage/features/etl/leads/tasks.py`
  - 更新导入路径：`from .etl import process_leads_etl`
  - `run_etl_leads()` 使用 `asyncio.to_thread()` 包装
  - 事件监听器使用 `@event_task_handler` 装饰器
  - 监听 `CUSTOMERS_ETL_COMPLETED` 事件
  - _需求: REQ-001, REQ-005_

## 阶段 5：其他 ETL 模块

- [ ] 5.1 重构 order_items ETL
  - 修改文件 `OneManage/features/etl/order_items/etl.py`
  - 使用 `core.database_sync.engine`
  - 使用 `BatchProcessor` 进行批量处理
  - 向量化字符串解析（如果可能）
  - _需求: REQ-001, REQ-002, REQ-003_

- [ ] 5.2 重构 process_items ETL
  - 修改文件 `OneManage/features/etl/process_items/etl.py`
  - 使用 `core.database_sync.engine`
  - 使用 `BatchProcessor` 进行批量处理
  - 向量化数据转换（计算 picture_print_quantity）
  - _需求: REQ-001, REQ-002, REQ-003, REQ-004_

- [ ] 5.3 重构 shop ETL
  - 修改文件 `OneManage/features/etl/shop/etl.py`
  - 使用 `core.database_sync.engine`
  - 向量化数据提取和去重
  - 使用 Polars `unique()` 和 `anti` join
  - _需求: REQ-001, REQ-002, REQ-004_

## 阶段 6：清理和优化

- [ ] 6.1 删除冗余文件
  - 删除 `OneManage/features/etl/event_tasks.py`（示例文件）
  - 删除 `OneManage/features/etl/models.py`（空文件）
  - 删除 `OneManage/features/etl/leads/etl_v2.py`（已重命名）
  - 删除 `OneManage/features/etl/orders/etl_no_polars_version.py`（旧版本）
  - 删除 `OneManage/features/etl/orders/etl_sql_version.py`（旧版本）
  - _需求: REQ-009_

- [ ] 6.2 统一事件监听器
  - 检查所有 `tasks.py` 文件
  - 确保所有事件监听器使用 `@event_task_handler` 装饰器
  - 确保所有事件主题使用 `events/subjects.py` 中的常量
  - 删除重复的版本检查、字段验证代码
  - 统一日志格式
  - _需求: REQ-005_

- [ ] 6.3 创建 ETL 模块文档
  - 创建文件 `OneManage/features/etl/README.md`
  - 说明 ETL 模块架构
  - 说明服务层的使用方法
  - 说明向量化最佳实践
  - 说明如何添加新的 ETL 模块
  - 说明如何使用 `core.database_sync`
  - 包含代码示例
  - _需求: REQ-010_
