# 数据导入模块重构设计文档

## 概述

本文档描述 `features/data_import` 模块的重构设计方案。重构采用清晰的分层架构，将代码职责明确划分为：Models（数据模型）、CRUD（数据访问）、Processor（业务逻辑）、Router（路由层）、Tasks（异步任务）。

## 架构设计

### 分层架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        Client / API                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Router Layer (router.py)                  │
│  - 路由定义                                                   │
│  - 参数验证                                                   │
│  - 依赖注入                                                   │
│  - 权限检查                                                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 Processor Layer (processor.py)               │
│  - 业务流程编排                                               │
│  - 文件处理                                                   │
│  - 数据验证                                                   │
│  - 报告生成                                                   │
│  - 事件发布                                                   │
└─────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
┌──────────────────────────┐  ┌──────────────────────────┐
│  Service Layer           │  │  CRUD Layer              │
│  (service.py)            │  │  (crud_async.py)         │
│  - ImportService         │  │  - 数据库查询            │
│  - 数据验证（表头）      │  │  - 数据库写入            │
│  - 数据验证（类型）      │  │  - 批量 Upsert           │
│  - 数据验证（字段值）    │  │  - 简单 CRUD             │
│  - 行数据映射到模型      │  │                          │
└──────────────────────────┘  └──────────────────────────┘
                    │                   │
                    └─────────┬─────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Models Layer (models.py)                  │
│  - 表模型 (table=True)                                        │
│  - Create 模型                                                │
│  - Update 模型                                                │
│  - Read 模型                                                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Database (PostgreSQL)                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    TaskIQ Layer (tasks.py)                   │
│  - 异步任务定义                                               │
│  - 调用 Processor                                             │
└─────────────────────────────────────────────────────────────┘
```

### 数据流向

#### 同步流程（文件上传）
```
Client → Router → Processor.process_file_upload → CRUD.create_task
                                                 → 保存文件
                                                 → 启动异步任务
                                                 → 返回任务信息
```

#### 异步流程（数据导入）
```
TaskIQ → Tasks.task_run_data_import → Processor.process_import_task
                                    → Processor.parse_file (Polars)
                                    → Service.validate_and_save_orders
                                        → Service.validate_headers
                                        → Service.validate_data_types
                                        → Service.validate_field_values
                                        → Service.map_row_to_model
                                        → CRUD.bulk_upsert_orders
                                    → Processor.generate_report
                                    → CRUD.update_task (更新状态)
                                    → Processor.publish_event (NATS)
                                    → Processor.cleanup_temp_file
```

## 职责划分说明

### Service vs Processor

**service.py（ImportService）**：
- **职责**：数据验证和转换的业务规则
- **关注点**：数据质量、业务规则验证
- **功能**：
  - 验证表头是否包含必填字段
  - 验证数据类型转换（字符串→数字、日期等）
  - 验证字段值（order_id 唯一性、金额负数警告等）
  - 将 DataFrame 行映射为 Order 模型
  - 执行完整的验证流程并调用 CRUD 保存
- **不负责**：文件 I/O、任务状态管理、事件发布

**processor.py**：
- **职责**：整体业务流程编排和基础设施操作
- **关注点**：流程控制、外部系统交互
- **功能**：
  - 文件上传处理（保存到临时目录）
  - 文件解析（Polars 读取 CSV/Excel）
  - 任务状态管理（PENDING → PROCESSING → COMPLETED/FAILED）
  - 报告生成（CSV 文件）
  - 事件发布（NATS）
  - 临时文件清理
  - 调用 service 进行数据验证和保存
- **不负责**：具体的数据验证规则

**简单类比**：
- **service.py** = 质检员（检查数据是否合格）
- **processor.py** = 生产线管理员（协调整个生产流程）

## 组件设计

### 1. Models Layer (models.py)

#### 设计原则
- 使用 SQLModel 单一模型定义
- 表模型使用 `table=True`
- API 模型继承表模型但不设置 `table=True`
- 使用现代联合类型语法 `| None`

#### 模型结构

```python
# 基础表模型
class DataImportTask(TimestampModel, table=True):
    __tablename__ = "data_import_tasks"
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    filename: str
    created_by: str
    status: DataImportTaskStatus
    # ... 其他字段

# Create 模型（用于 API 请求）
class DataImportTaskCreate(SQLModel):
    filename: str
    created_by: str

# Update 模型（用于部分更新）
class DataImportTaskUpdate(SQLModel):
    status: DataImportTaskStatus | None = None
    total_rows_in_file: int | None = None
    # ... 其他可选字段

# Read 模型（用于 API 响应）
class DataImportTaskRead(SQLModel):
    id: uuid.UUID
    filename: str
    created_by: str
    status: DataImportTaskStatus
    created_at: datetime
    updated_at: datetime
    # ... 其他字段
    
    model_config = {"from_attributes": True}
```

#### Order 模型设计
```python
class Order(TimestampModel, table=True):
    __tablename__ = "raw_orders"
    # 继承 id, created_at, updated_at
    data_import_task_id: uuid.UUID | None
    order_id: str = Field(unique=True, index=True)
    version: int = Field(default=1)
    # ... 业务字段

class OrderCreate(SQLModel):
    order_id: str
    # ... 业务字段（不包含 id, timestamps）

class OrderUpdate(SQLModel):
    order_id: str | None = None
    # ... 所有字段可选

class OrderRead(SQLModel):
    id: int
    order_id: str
    version: int
    created_at: datetime
    updated_at: datetime
    # ... 业务字段
    
    model_config = {"from_attributes": True}
```

### 2. CRUD Layer (crud_async.py)

#### 设计原则
- 遵循 slim-template 格式
- 简洁的函数签名
- 使用 `model_validate` 进行模型转换
- 查询不到记录时抛出 `ValueError`
- 每个函数包含中文文档字符串

#### 函数设计

```python
async def get_task(db: AsyncSession, task_id: uuid.UUID) -> DataImportTask | None:
    """根据任务ID查询导入任务"""
    result = await db.execute(
        select(DataImportTask).where(DataImportTask.id == task_id)
    )
    return result.scalars().first()

async def create_task(db: AsyncSession, payload: DataImportTaskCreate) -> DataImportTask:
    """创建数据导入任务"""
    task = DataImportTask.model_validate(payload)
    db.add(task)
    await db.flush()
    await db.refresh(task)
    return task

async def update_task(
    db: AsyncSession, 
    task_id: uuid.UUID, 
    payload: DataImportTaskUpdate
) -> DataImportTask:
    """更新数据导入任务"""
    task = await get_task(db, task_id)
    if task is None:
        raise ValueError(f"任务不存在: {task_id}")
    
    data = payload.model_dump(exclude_unset=True)
    for key, value in data.items():
        setattr(task, key, value)
    
    db.add(task)
    await db.flush()
    await db.refresh(task)
    return task

async def get_order(db: AsyncSession, order_id: str) -> Order | None:
    """根据订单ID查询订单"""
    result = await db.execute(
        select(Order).where(Order.order_id == order_id)
    )
    return result.scalars().first()

async def get_orders(
    db: AsyncSession, 
    limit: int = 100, 
    offset: int = 0
) -> list[Order]:
    """获取订单列表"""
    result = await db.execute(
        select(Order)
        .order_by(Order.updated_at.desc())
        .limit(limit)
        .offset(offset)
    )
    return result.scalars().all()

async def bulk_upsert_orders(
    db: AsyncSession, 
    orders: list[Order]
) -> tuple[int, int, int]:
    """批量插入或更新订单
    
    业务规则：
    - 新订单：直接插入，version=1
    - 现有订单：只有当新数据的completion_date比现有数据更新时才允许更新
    - 更新时：version递增
    
    返回：(inserted_count, updated_count, skipped_count)
    """
    # 保留原有的 upsert 逻辑
    # 使用 PostgreSQL INSERT ... ON CONFLICT DO UPDATE
    # 详细实现见原 crud.py
```

### 3. Processor Layer (processor.py)

#### 设计原则
- 核心业务逻辑编排
- 可被 router 和 tasks 复用
- 统一异常处理
- 记录详细日志

#### 函数设计

##### 文件上传处理
```python
async def process_file_upload(
    file: UploadFile,
    user_id: str,
    db: AsyncSession
) -> tuple[DataImportTask, str]:
    """处理文件上传并创建导入任务
    
    Args:
        file: 上传的文件对象
        user_id: 用户ID
        db: 数据库会话
        
    Returns:
        (task, file_path): 任务对象和保存的文件路径
        
    Raises:
        ValueError: 文件格式不支持
    """
    # 1. 验证文件格式
    if not file.filename.endswith((".csv", ".xlsx", ".xls")):
        raise ValueError("只支持CSV或Excel文件")
    
    # 2. 创建任务记录
    task_create = DataImportTaskCreate(
        filename=file.filename,
        created_by=user_id
    )
    task = await crud.create_task(db, task_create)
    
    # 3. 保存文件
    file_path = save_upload_file(file, task.id)
    
    logger.info(f"文件上传成功: task_id={task.id}, file={file.filename}")
    
    return task, file_path
```

##### 文件解析
```python
def parse_file(file_path: str) -> pl.DataFrame:
    """解析上传文件为 Polars DataFrame
    
    Args:
        file_path: 文件路径
        
    Returns:
        pl.DataFrame: 解析后的数据框
        
    Raises:
        ValueError: 文件格式不支持或解析失败
    """
    try:
        if file_path.endswith(".csv"):
            return pl.read_csv(file_path)
        elif file_path.endswith((".xlsx", ".xls")):
            return pl.read_excel(file_path, engine="calamine")
        else:
            raise ValueError(f"不支持的文件格式: {file_path}")
    except Exception as e:
        logger.error(f"文件解析失败: {file_path}, 错误: {e}")
        raise ValueError(f"文件解析失败: {str(e)}")
```

##### 报告生成
```python
def generate_report(
    task_id: uuid.UUID,
    processed_orders: list[Order]
) -> str:
    """生成导入报告
    
    Args:
        task_id: 任务ID
        processed_orders: 已处理的订单列表
        
    Returns:
        str: 报告文件路径
    """
    report_filename = f"{task_id}_report.csv"
    report_path = os.path.join(REPORT_DIR, report_filename)
    
    with open(report_path, "w", encoding="utf-8") as f:
        f.write("order_id,status,message,version\n")
        for order in processed_orders:
            f.write(f"{order.order_id},成功,,{order.version}\n")
    
    logger.info(f"报告生成成功: {report_path}")
    return report_path
```

##### 核心导入流程
```python
async def process_import_task(task_id: uuid.UUID, file_path: str) -> None:
    """统一的导入任务处理逻辑
    
    流程：
    1. 更新状态为 PROCESSING
    2. 解析文件 (Polars)
    3. 验证并保存数据 (ImportService)
    4. 生成报告
    5. 更新最终状态 (COMPLETED/FAILED)
    6. 发布完成事件
    7. 清理临时文件
    
    Args:
        task_id: 任务ID
        file_path: 文件路径
    """
    logger.info(f"[Processor] 开始处理任务 {task_id}")
    
    status = "success"
    processed_items = 0
    
    try:
        async with AsyncSessionFactory() as db:
            try:
                # 1. 更新状态为 PROCESSING
                await crud.update_task(
                    db,
                    task_id,
                    DataImportTaskUpdate(
                        status=DataImportTaskStatus.PROCESSING,
                        started_at=datetime.utcnow()
                    )
                )
                await db.commit()
                
                # 2. 解析文件
                df = parse_file(file_path)
                await crud.update_task(
                    db,
                    task_id,
                    DataImportTaskUpdate(total_rows_in_file=len(df))
                )
                await db.commit()
                
                # 3. 验证并保存数据
                processed_orders, errors, warnings, stats = \
                    await service.validate_and_save_orders(df, db, task_id)
                
                # 4. 确定最终状态
                final_status = (
                    DataImportTaskStatus.COMPLETED 
                    if not errors 
                    else DataImportTaskStatus.FAILED
                )
                
                # 5. 生成报告
                report_path = None
                if final_status == DataImportTaskStatus.COMPLETED:
                    if stats.get("db_processed_count", 0) > 0:
                        report_path = generate_report(task_id, processed_orders)
                
                # 6. 更新最终状态
                update_data = DataImportTaskUpdate(
                    status=final_status,
                    finished_at=datetime.utcnow(),
                    error_message="; ".join(errors) if errors else None,
                    report_path=report_path,
                    **stats
                )
                await crud.update_task(db, task_id, update_data)
                await db.commit()
                
                processed_items = stats.get("db_processed_count", 0)
                
            except Exception as e:
                await db.rollback()
                logger.error(f"[Processor] 任务处理失败: {e}")
                
                # 更新为失败状态
                await crud.update_task(
                    db,
                    task_id,
                    DataImportTaskUpdate(
                        status=DataImportTaskStatus.FAILED,
                        finished_at=datetime.utcnow(),
                        error_message=str(e)
                    )
                )
                await db.commit()
                status = "failed"
                
    finally:
        # 7. 清理临时文件
        cleanup_temp_file(file_path)
        
        # 8. 发布事件
        await publish_import_completed_event(
            task_id, status, file_path, processed_items
        )
        
        logger.info(f"[Processor] 任务完成: status={status}")
```

##### 辅助函数
```python
def save_upload_file(file: UploadFile, task_id: uuid.UUID) -> str:
    """保存上传文件到临时目录"""
    file_path = os.path.join(UPLOAD_DIR, f"{task_id}_{file.filename}")
    with open(file_path, "wb") as buffer:
        buffer.write(file.file.read())
    return file_path

def cleanup_temp_file(file_path: str) -> None:
    """清理临时文件"""
    if os.path.exists(file_path):
        try:
            os.remove(file_path)
            logger.info(f"临时文件已删除: {file_path}")
        except Exception as e:
            logger.error(f"删除临时文件失败: {e}")

async def publish_import_completed_event(
    task_id: uuid.UUID,
    status: str,
    file_path: str,
    processed_items: int
) -> None:
    """发布导入完成事件"""
    from events.publisher import publish_event
    from events.subjects import DATA_IMPORT_COMPLETED
    from events.schemas import DataImportCompletedEvent
    
    await publish_event(
        DATA_IMPORT_COMPLETED,
        DataImportCompletedEvent(
            import_id=str(task_id),
            status=status,
            file_path=file_path,
            processed_items=processed_items,
        ).model_dump(mode="json"),
    )
```

### 4. Router Layer (router.py)

#### 设计原则
- 瘦路由：只负责路由定义
- 参数验证和依赖注入
- 调用 processor 处理业务逻辑
- 每个函数不超过 15 行

#### 路由设计

```python
@router.post("/orders", response_model=DataImportTaskRead)
async def upload_and_import(
    file: UploadFile = File(...),
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(require_permission(DATA_IMPORT_PERMISSION)),
):
    """文件上传并启动异步导入任务"""
    # 1. 处理文件上传
    task, file_path = await processor.process_file_upload(
        file, str(current_user.id), db
    )
    await db.commit()
    
    # 2. 启动异步任务
    await task_run_data_import.kiq(str(task.id), file_path)
    
    return task

@router.get("/orders/{task_id}/status", response_model=DataImportTaskRead)
async def check_status(
    task_id: uuid.UUID,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(require_permission(DATA_IMPORT_PERMISSION)),
):
    """查询任务状态"""
    task = await crud.get_task(db, task_id)
    if not task:
        raise HTTPException(status_code=404, detail="任务不存在")
    return task

@router.get("/orders/{task_id}/report")
async def download_report(
    task_id: uuid.UUID,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(require_permission(DATA_IMPORT_PERMISSION)),
):
    """下载导入报告"""
    task = await crud.get_task(db, task_id)
    if not task or not task.report_path:
        raise HTTPException(status_code=404, detail="报告不存在")
    
    if not os.path.exists(task.report_path):
        raise HTTPException(status_code=404, detail="报告文件不存在")
    
    return FileResponse(
        task.report_path,
        media_type="text/csv",
        filename=os.path.basename(task.report_path)
    )
```

### 5. Tasks Layer (tasks.py)

#### 设计原则
- 只负责任务定义
- 调用 processor 处理业务逻辑
- 保持简洁，不超过 10 行

#### 任务设计

```python
@broker.task
async def task_run_data_import(task_id: str, file_path: str) -> None:
    """数据导入异步任务
    
    Args:
        task_id: 任务ID（字符串格式）
        file_path: 文件路径
    """
    logger.info(f"[TaskIQ] 开始执行导入任务: {task_id}")
    
    await processor.process_import_task(uuid.UUID(task_id), file_path)
    
    logger.info(f"[TaskIQ] 导入任务完成: {task_id}")
```

## 数据模型

### DataImportTask 状态机

```
PENDING ──────▶ PROCESSING ──────▶ COMPLETED
                    │
                    └──────▶ FAILED
```

### Order 版本控制

- 新订单：`version = 1`
- 更新订单：`version = version + 1`
- 更新条件：`new.completion_date > old.completion_date`

## 错误处理

### 异常层级

1. **Processor 层**：捕获所有异常，更新任务状态为 FAILED
2. **Router 层**：转换为 HTTPException
3. **CRUD 层**：抛出 ValueError（记录不存在）

### 错误日志

使用 loguru 记录：
- INFO：正常流程节点
- WARNING：可恢复的异常
- ERROR：业务异常
- EXCEPTION：系统异常（包含堆栈）

## 测试策略

### 单元测试（可选）
- CRUD 层函数测试
- Processor 辅助函数测试

### 集成测试（可选）
- API 端点测试
- 完整导入流程测试

## 性能考虑

### 批量操作
- 使用 PostgreSQL `INSERT ... ON CONFLICT` 进行 upsert
- 分批处理（chunk_size=500）避免参数过多

### 异步处理
- 文件上传：同步（<1秒）
- 数据导入：异步（>2秒）
- 使用 TaskIQ + NATS 解耦

### 内存优化
- Polars 流式处理大文件
- 及时清理临时文件

## 向后兼容

### API 兼容
- 保持所有端点路径不变
- 保持请求/响应格式不变

### 数据库兼容
- 不修改表结构
- 不修改字段类型

### 事件兼容
- 保持 NATS 事件格式不变
- 保持事件主题不变

## 部署考虑

### 文件结构变更
```
features/data_import/
├── models.py          # 合并后的模型（新）
├── crud_async.py      # 重构后的 CRUD（新）
├── processor.py       # 重构后的业务逻辑（修改）
├── router.py          # 瘦路由（修改）
├── tasks.py           # 简化的任务（修改）
├── service.py         # 保持不变
├── raw_order_importer.py  # 保持不变（可选删除）
└── schemas.py         # 删除（已合并到 models.py）
```

### 迁移步骤
1. 创建新文件（models.py 合并版、crud_async.py）
2. 修改现有文件（processor.py、router.py、tasks.py）
3. 更新导入语句
4. 删除 schemas.py
5. 运行测试验证
6. 部署上线

## 风险评估

### 低风险
- ✅ 合并 models 和 schemas
- ✅ 重构 CRUD 格式
- ✅ 简化 tasks

### 中风险
- ⚠️ 重构 processor（需要仔细测试）
- ⚠️ 修改 router 调用方式

### 缓解措施
- 保持 API 接口不变
- 保持数据库结构不变
- 充分的代码审查
- 渐进式部署（如有需要）
