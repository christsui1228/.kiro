---
inclusion: always
---

# Spec 开发规范

## 语言规范
- 所有文档必须使用简体中文
- 代码注释使用简体中文
- 变量名和函数名使用英文，但注释说明用简体中文

## 命令行运行规范
- 所有 Python 相关命令必须使用 `pdm run` 前缀
- 示例：`pdm run python script.py`、`pdm run pytest`、`pdm run black .`

## Spec 文件结构规范
- 需求文档：`requirements.md` - 使用 EARS 格式的验收标准
- 设计文档：`design.md` - 包含架构图、组件设计、数据模型
- 任务文档：`tasks.md` - 可执行的编码任务，引用具体需求

## 任务编写规范
- 每个任务必须是具体的编码活动
- 任务描述要明确，包含要修改的文件
- 可选任务用 `*` 标记（主要是测试相关）
- 每个任务必须引用对应的需求编号

## 目录命名规范
- 文件名保持英文，便于系统兼容性

## 文档创建规范

### 禁止创建的文档类型
为了提高执行速度和节省 token，在任务执行过程中**严格禁止**创建以下类型的 Markdown 文档：

❌ **禁止创建**：
- 实施总结文档（`IMPLEMENTATION_SUMMARY.md`、`TASK_X_IMPLEMENTATION_SUMMARY.md`）
- 验证清单文档（`VERIFICATION_CHECKLIST.md`、`TASK_X_VERIFICATION.md`）
- 集成指南文档（`INTEGRATION_GUIDE.md`）
- 使用说明文档（`README.md`、`USER_GUIDE.md`）
- 测试指南文档（`TEST_GUIDE.md`）
- 性能优化文档（`PERFORMANCE_OPTIMIZATION.md`）
- 故障排查文档（`TROUBLESHOOTING.md`）
- 任何其他总结性、说明性的 MD 文档

### 允许的文档操作
✅ **仅允许**：
- 修改 Spec 核心文档（`requirements.md`、`design.md`、`tasks.md`）
- 修改项目根目录的 `README.md`（仅在明确要求时）
- 创建代码注释和 docstring（在代码文件内部）

### 执行原则
- 任务完成后，直接向用户口头汇报结果，不创建总结文档
- 验证结果直接在对话中说明，不创建验证清单
- 使用说明通过代码注释和 docstring 提供
- 集成方式在对话中解释，不创建单独文档

## 开发流程规范

### 核心原则
在进行任何代码修改之前，必须遵循以下开发流程：

#### 1. 需求分析阶段
- 必须先完整理解用户需求
- 明确功能范围和边界条件
- 识别潜在的技术风险和依赖关系

#### 2. 方案设计阶段
- 提供多个技术方案选项
- 详细说明每个方案的优缺点
- 包含性能、可维护性、扩展性的考量
- **重要：未经用户明确同意，不得开始编写代码**

#### 3. 实施前确认
- 向用户展示完整的实施计划
- 说明将要修改的文件和主要变更点
- 获得用户明确的"开始实施"确认后才能进行代码修改

#### 4. 渐进式实施
- 采用小步骤、可回滚的方式进行修改
- 每个重要节点都要向用户汇报进度
- 遇到问题时立即停止并寻求用户指导

### 禁止行为
- 不得在方案讨论阶段直接开始编码
- 不得在未获得明确同意的情况下修改现有代码
- 不得跳过设计阶段直接进入实施
- 不得在用户提出问题时立即给出代码示例（除非明确要求）
- **不得擅自创建或修改文件，必须获得用户明确指令**

### 文件操作确认规则

#### 需要明确确认的操作
以下操作必须先展示方案并获得用户明确同意：
- 创建新文件（代码文件、配置文件、文档等）
- 修改现有文件
- 删除文件
- 重命名或移动文件

#### 明确指令识别
只有当用户使用以下明确动词时才可直接执行：
- ✅ "创建"、"生成"、"写入"、"添加"
- ✅ "修改"、"更新"、"改为"
- ✅ "帮我创建"、"直接创建"

#### 需要先展示方案的情况
当用户使用以下表述时，应先展示方案：
- ❌ "我需要一个..."、"应该有个..."
- ❌ "设计一个..."、"规划一个..."
- ❌ "给我一个...的方案"

#### 标准确认流程
1. **展示方案**：列出文件路径、主要内容结构、关键代码片段
2. **说明影响**：解释这个文件的作用和影响范围
3. **询问确认**：明确询问"是否创建/修改这个文件？"
4. **等待回复**：获得明确同意后才执行操作

#### 示例对比
```
❌ 用户："我需要一个 API 文档"
   AI：[直接创建文件] ← 错误

✅ 用户："我需要一个 API 文档"
   AI：展示文档大纲 → 询问是否创建 → 等待确认

✅ 用户："帮我创建一个 API 文档"
   AI：[直接创建文件] ← 正确
```

### 沟通要求
- 使用简体中文进行所有交流
- 保持专业但友好的语调
- 优先提供思路和方案，而非直接的代码实现
- 在每个阶段都要征求用户的意见和确认

### 方案设计原则
- **根治性方案**: 不提供头痛医头脚痛医脚的临时解决方案
- **现代化优先**: 优先采用最新的技术和最佳实践
- **系统性思考**: 从架构层面解决问题，而非局部修补
- **可扩展性**: 方案必须考虑未来的扩展和维护需求
- **技术前瞻性**: 选择有长期发展前景的技术方案

### 异常处理
如果用户明确要求跳过某些流程步骤，需要：
1. 确认用户完全理解跳过的风险
2. 记录用户的明确授权
3. 在实施过程中保持额外的谨慎

### 方案质量标准
- **避免临时方案**: 不提供仅解决表面问题的方案
- **追求根本解决**: 深入分析问题根源，提供系统性解决方案
- **现代化技术**: 优先使用最新稳定版本和现代化架构模式
- **最佳实践**: 遵循行业最佳实践和设计模式
- **长远考虑**: 方案要能适应未来 2-3 年的技术发展趋势

#### 示例对比
❌ **临时方案**: "可以用 try-catch 包装一下这个错误"
✅ **根治方案**: "建议重构错误处理架构，使用统一的异常处理中间件和结构化错误响应"

❌ **过时方案**: "可以用 requests 库同步调用"
✅ **现代方案**: "使用 aiohttp 异步客户端，配合 TaskIQ 任务队列处理"

## 任务管理规范

### 混合管理策略
采用 Spec 任务和模块任务的混合管理方式：

#### Spec 任务管理
- **位置**: `.kiro/specs/{功能名}/tasks.md`
- **适用场景**: 
  - 需要完整需求分析和设计的新功能
  - 复杂的功能开发项目
  - 需要跨模块协作的任务
- **流程**: requirements → design → tasks → 实施
- **示例**: PSD图层预览功能、用户认证系统

#### 模块任务管理  
- **位置**: `features/{模块名}/task-management.md`
- **适用场景**:
  - 模块内部的技术债务和优化
  - 不需要完整 spec 流程的小任务
  - 模块维护和重构任务
- **格式**: 使用 ID-XXX 编号系统
- **示例**: 性能优化、算法改进、依赖升级

#### 任务状态同步
- Spec 任务完成后，同步更新模块 task-management.md 状态
- 模块任务如需升级为 Spec，创建对应的 spec 目录并更新引用
- 保持两个系统的任务状态一致性

## 同步 vs 异步决策规范

### 决策标准：2秒阈值原则

**同步处理（优先）**：
- ✅ 执行时间 < 2秒
- ✅ 需要立即返回结果给用户
- ✅ 简单的数据库查询和更新
- ✅ 轻量级数据转换和验证

**异步处理（TaskIQ）**：
- ✅ 执行时间 > 2秒
- ✅ 计算密集型任务（图像处理、数据分析）
- ✅ IO 密集型任务（文件上传、外部 API 调用）
- ✅ 批量数据处理
- ✅ 可以延迟执行的任务

### 典型场景示例

**同步场景**：
```python
# ✅ 用户登录验证（<100ms）
@router.post("/login")
async def login(credentials: LoginRequest):
    user = await auth_service.verify_credentials(credentials)
    return {"token": generate_token(user)}

# ✅ 简单数据查询（<500ms）
@router.get("/users/{user_id}")
async def get_user(user_id: str):
    return await user_service.get_by_id(user_id)

# ✅ 数据验证和创建（<1秒）
@router.post("/orders")
async def create_order(order: OrderCreate):
    validated_order = await order_service.validate_and_create(order)
    return validated_order
```

**异步场景**：
```python
# ✅ PSD 文件解析（5-30秒）
@router.post("/psd/parse")
async def parse_psd(file: UploadFile):
    task_id = str(uuid.uuid4())
    await task_parse_psd.kiq(file_path, task_id)
    return {"task_id": task_id, "status": "processing"}

# ✅ 批量数据导入（分钟级）
@router.post("/data/import")
async def import_data(file: UploadFile):
    task_id = await task_import_excel.kiq(file_path)
    return {"task_id": task_id, "status": "queued"}

# ✅ 图像生成（10-60秒）
@router.post("/images/generate")
async def generate_image(params: ImageParams):
    task_id = await task_generate_background.kiq(params)
    return {"task_id": task_id, "status": "queued"}
```

### 混合模式：同步入口 + 异步处理

```python
# 快速响应 + 后台处理
@router.post("/orders")
async def create_order(order: OrderCreate):
    # 1. 同步：快速验证和创建订单（<1秒）
    db_order = await order_service.create(order)
    
    # 2. 异步：发送邮件、更新库存等（后台处理）
    await task_send_order_confirmation.kiq(db_order.id)
    await task_update_inventory.kiq(db_order.items)
    
    # 3. 立即返回订单信息
    return db_order
```

### 性能优化策略

**数据库操作**：
- 简单查询：同步（使用索引，<100ms）
- 复杂聚合：考虑异步或缓存
- 批量写入：异步（使用 Polars + 分批处理）

**外部 API 调用**：
- 关键路径：同步 + 超时控制（2秒超时）
- 非关键路径：异步（邮件、通知、日志上报）

**文件处理**：
- 小文件（<1MB）：同步读取和验证
- 大文件（>1MB）：异步处理
- 文件上传：同步保存 + 异步处理

## TaskIQ + NATS 架构规范

项目采用 TaskIQ + NATS 双流架构进行异步任务和事件处理。

### 核心概念
- **TASKS流**: 命令任务，使用 `@broker.task` 装饰器，1:1 语义
- **EVENTS流**: 事件处理器，不使用装饰器，1:N 语义
- **三个独立服务**: FastAPI 服务器、TaskIQ Worker、事件消费者

### 命名规范快速参考
```python
# 命令任务函数名
task_parse_psd()
task_generate_background()

# 事件处理器函数名
on_psd_parse_completed_publish_preview()
on_data_import_completed_etl()

# 事件主题格式
"events.{domain}.{action}.{status}"
"events.psd.parse_export.completed"
"events.data_import.started"
```

### 服务启动
```bash
# 开发环境启动顺序
docker run -p 4222:4222 nats:latest --jetstream  # 1. NATS
pdm run uvicorn main:app --reload                # 2. API
pdm run taskiq worker core.broker:broker --workers 1  # 3. Worker
pdm run python scripts/run_event_consumer.py    # 4. 事件消费者
```

### 详细规范参考
完整的架构设计、代码示例和最佳实践请参考：
- **backend-architecture.md** - Tasks Layer 分层架构详解
- **backend-tech-stack.md** - TaskIQ + NATS 配置和双流架构图

### 关键原则
- ✅ 命令任务用于执行操作并返回结果
- ✅ 事件用于通知状态变化和解耦模块
- ❌ 不要使用 `@broker.task` 装饰事件处理器
- ❌ 不要在事件处理器中执行长时间任务（>2秒）