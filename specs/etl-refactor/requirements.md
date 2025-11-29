# ETL 模块重构需求文档

## 项目概述

**项目名称**：ETL 模块重构  
**项目目标**：统一 ETL 模块架构，提升代码质量和性能  
**优先级**：P1（高优先级）  
**预计工时**：21-29 小时

## 业务背景

当前 ETL 模块存在以下问题：
1. 架构不统一：同步/异步混用，代码风格不一致
2. 性能问题：大量逐行查询数据库，未充分利用向量化操作
3. 代码重复：每个子模块都重复实现数据库连接、批处理、错误处理
4. 事件监听器重复：每个模块都重复写版本检查、状态判断逻辑

## 验收标准（EARS 格式）

### REQ-001：统一架构模式
**WHEN** 执行任何 ETL 任务  
**THE SYSTEM SHALL** 使用 `asyncio.to_thread()` 包装同步 Polars 代码  
**AND** 通过 TaskIQ + NATS 进行任务调度和事件通信

**验收标准**：
- 所有 ETL 任务都使用 `asyncio.to_thread()` 包装
- 所有 ETL 任务都通过 TaskIQ 队列执行
- 所有 ETL 完成后都发布 NATS 事件

### REQ-002：复用现有基础设施
**WHEN** ETL 模块需要数据库连接  
**THE SYSTEM SHALL** 使用 `core.database_sync.engine`  
**AND** 不重复创建数据库引擎

**验收标准**：
- 所有 ETL 模块导入 `from core.database_sync import engine`
- 删除所有自定义的 `_get_engine()` 函数
- 数据库连接池配置统一管理

### REQ-003：创建可复用的服务层
**WHEN** 多个 ETL 模块需要相同功能  
**THE SYSTEM SHALL** 提供统一的服务层工具  
**AND** 消除代码重复

**验收标准**：
- 创建 `features/etl/services/batch.py` 提供批处理服务
- 创建 `features/etl/services/vectorized_queries.py` 提供向量化查询工具
- 所有 ETL 模块使用服务层工具，不重复实现

### REQ-004：向量化数据处理
**WHEN** 处理批量数据  
**THE SYSTEM SHALL** 使用 Polars 向量化操作  
**AND** 避免逐行处理

**验收标准**：
- 数据提取：使用批量查询，一次性获取所有依赖数据
- 数据转换：使用 Polars 表达式，不使用 Python 循环
- 数据加载：使用批量 INSERT/UPSERT，不逐行插入

### REQ-005：统一事件监听器
**WHEN** ETL 模块监听事件  
**THE SYSTEM SHALL** 使用 `@event_task_handler` 装饰器  
**AND** 使用 `events/subjects.py` 中定义的事件主题

**验收标准**：
- 所有事件监听器使用 `@event_task_handler` 装饰器
- 删除重复的版本检查、字段验证代码
- 统一使用 `events/subjects.py` 中的事件常量

### REQ-006：重命名 leads ETL 文件
**WHEN** 访问 leads ETL 模块  
**THE SYSTEM SHALL** 使用 `etl.py` 作为主文件名  
**AND** 删除 `etl_v2.py` 文件

**验收标准**：
- `features/etl/leads/etl_v2.py` 重命名为 `etl.py`
- 所有导入路径更新为 `from .etl import ...`
- 删除旧的 `etl.py` 文件（如果存在）

### REQ-007：性能优化
**WHEN** 处理 1000 条订单数据  
**THE SYSTEM SHALL** 在指定时间内完成处理  
**AND** 性能不低于重构前

**验收标准**：
- 订单 ETL：1000 条 < 5 秒
- 客户 ETL：1000 条 < 3 秒
- 商机 ETL：1000 条 < 10 秒（当前 15 秒，需优化）

### REQ-008：数据准确性
**WHEN** 重构后执行 ETL  
**THE SYSTEM SHALL** 保持数据准确性  
**AND** 结果与重构前一致

**验收标准**：
- 使用相同测试数据，重构前后结果一致
- 所有字段映射正确
- 统计字段计算准确

### REQ-009：清理冗余代码
**WHEN** 重构完成  
**THE SYSTEM SHALL** 删除所有冗余文件  
**AND** 保持代码库整洁

**验收标准**：
- 删除 `features/etl/event_tasks.py`（示例文件）
- 删除 `features/etl/models.py`（空文件）
- 删除 `features/etl/leads/etl_v2.py`（已重命名）

### REQ-010：文档更新
**WHEN** 重构完成  
**THE SYSTEM SHALL** 提供完整的文档  
**AND** 说明新架构的使用方法

**验收标准**：
- 创建 `features/etl/README.md` 说明架构
- 文档包含服务层使用指南
- 文档包含向量化最佳实践

## 非功能性需求

### NFR-001：可维护性
- 代码遵循 DRY 原则，无重复逻辑
- 统一的错误处理和日志格式
- 清晰的模块职责划分

### NFR-002：可扩展性
- 服务层设计支持未来添加新的 ETL 模块
- 向量化查询工具支持扩展新的查询类型
- 批处理服务支持自定义处理逻辑

### NFR-003：可测试性
- 服务层工具函数易于单元测试
- ETL 逻辑与任务包装分离，便于测试
- 向量化操作便于用小数据集验证

### NFR-004：兼容性
- 保持与现有 TaskIQ + NATS 架构兼容
- 保持与现有数据库模型兼容
- 保持与现有事件系统兼容

## 约束条件

1. **技术栈约束**：
   - 必须使用 Polars 进行数据处理
   - 必须使用 TaskIQ + NATS 进行任务调度
   - 必须使用 `core.database_sync` 进行数据库操作

2. **性能约束**：
   - 不能降低现有 ETL 性能
   - 商机 ETL 需要提升性能（从 15 秒降到 10 秒）

3. **兼容性约束**：
   - 不能破坏现有的事件发布/订阅机制
   - 不能改变数据库表结构
   - 不能改变 API 接口

4. **开发约束**：
   - 遵循项目的 Spec 开发流程
   - 遵循项目的代码规范（中文注释、PDM 管理）
   - 不引入新的外部依赖

## 风险评估

### 风险 1：向量化可能增加代码复杂度
**影响**：中  
**概率**：中  
**缓解措施**：
- 先在测试环境验证
- 保留原有逻辑作为备份
- 逐步迁移，不一次性全改

### 风险 2：性能可能不如预期
**影响**：中  
**概率**：低  
**缓解措施**：
- 添加性能监控
- 对比重构前后的耗时
- 如果性能下降，回退到原有方案

### 风险 3：数据准确性问题
**影响**：高  
**概率**：低  
**缓解措施**：
- 编写数据对比脚本
- 在测试环境用真实数据验证
- 灰度发布（先处理部分数据）

## 成功标准

1. ✅ 所有验收标准通过
2. ✅ 性能达到或超过预期
3. ✅ 代码审查通过
4. ✅ 测试环境验证通过
5. ✅ 文档完整且准确

## 参考文档

- `events/EVENT_TASKS_README.md` - 事件任务模式说明
- `events/event_task_utils.py` - 事件任务工具
- `core/database_sync.py` - 同步数据库模块
- `.kiro/steering/backend-tech-stack.md` - 后端技术栈规范
- `.kiro/steering/spec-conventions.md` - Spec 开发规范
