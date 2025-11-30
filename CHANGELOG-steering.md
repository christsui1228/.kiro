# Steering 规则重命名记录

## 📅 2025-10-20

### 🔄 重命名操作

为了更清晰地区分前后端规则，对 steering 目录下的文件进行了重命名：

#### 后端相关文件
| 旧文件名 | 新文件名 | 说明 |
|---------|---------|------|
| `tech-stack.md` | `backend-tech-stack.md` | 后端技术栈规范 |
| `code-style.md` | `backend-code-style.md` | Python 代码风格 |
| `deployment.md` | `backend-deployment.md` | 后端部署配置 |
| `environment-config.md` | `backend-environment-config.md` | 后端环境变量配置 |

#### 前端相关文件
| 文件名 | 说明 |
|-------|------|
| `frontend-tech-stack.md` | 前端技术栈规范（新增） |

#### 通用规则文件（未改动）
- `spec-conventions.md` - Spec 开发规范
- `development-workflow.md` - 开发流程策略
- `task-management.md` - 任务管理规范
- `spec-task-scope.md` - Spec 任务范围
- `vibe-mode.md` - Vibe 模式配置

### 📝 配置文件更新

#### `.kiro/settings/vibe-config.json`
```json
"steeringRules": [
  "backend-tech-stack",      // 原 tech-stack
  "frontend-tech-stack",     // 新增
  "backend-code-style",      // 原 code-style
  "spec-conventions"
]
```

#### `.kiro/README-steering.md`
- 更新了所有文件名引用
- 更新了示例代码
- 更新了规则应用矩阵

### ✅ 重命名原因

1. **清晰的职责划分** - 明确区分前后端规则
2. **便于维护** - 文件名即说明用途
3. **避免混淆** - 在 monorepo 中管理多个项目
4. **符合规范** - 统一的命名风格

### 🎯 影响范围

- ✅ 文件重命名完成
- ✅ vibe-config.json 更新完成
- ✅ README-steering.md 更新完成
- ✅ 所有引用已更新

### 📌 注意事项

如果你在其他地方（如文档、脚本）引用了旧的文件名，需要手动更新：
- `tech-stack.md` → `backend-tech-stack.md`
- `code-style.md` → `backend-code-style.md`
- `deployment.md` → `backend-deployment.md`
- `environment-config.md` → `backend-environment-config.md`

---

## 📅 2025-11-30

### 🎉 前端规范体系完善

新增和完善了完整的前端开发规范体系：

#### 新增文件
| 文件名 | 说明 |
|-------|------|
| `frontend-architecture.md` | 前端架构规范（组件设计、Composables、状态管理） |
| `frontend-code-style.md` | 前端代码风格规范（完整版） |

#### 完善文件
| 文件名 | 更新内容 |
|-------|---------|
| `frontend-tech-stack.md` | 添加环境变量、HTTP 配置、构建配置、开发工具等 |
| `backend-environment-config.md` | 改为嵌套类配置设计，参考 OneManage 实现 |
| `backend-architecture.md` | 分离异步/同步数据库会话（database_async.py / database_sync.py） |

### 📋 前端规范详细内容

#### `frontend-architecture.md`
- 完整的目录结构规范
- 组件设计原则（单一职责、组件分类）
- Composables 设计模式（usePolling、useTable、useDialog）
- 状态管理策略（Pinia Store 设计）
- 数据流设计（单向数据流）
- 错误边界处理
- 性能优化策略
- 测试策略

#### `frontend-code-style.md`
- script setup 语法规范
- 响应式数据命名
- 函数规范（命名、JSDoc、异步处理）
- 常量定义和枚举对象
- 错误处理模式
- 组件规范（拆分、Props、Emits）
- 样式规范（Scoped、类名命名）
- **API 调用规范**（统一封装、错误处理）
- **状态管理规范**（Pinia Store）
- **表单处理规范**（验证、提交）
- **列表渲染规范**（key、空状态）
- **路由使用规范**（编程式导航）
- **工具函数规范**（日期、文件大小格式化）

#### `frontend-tech-stack.md`
- 核心技术栈（Vue 3、Naive UI、Vite、Pinia）
- **环境变量配置**（命名、使用）
- **HTTP 客户端配置**（Axios 拦截器）
- **路由配置**（定义、守卫）
- **构建配置**（Vite、代码分割）
- **开发工具**（VSCode 插件、ESLint、Prettier）
- **性能优化**（懒加载、图片优化）
- **调试技巧**
- **部署规范**

### 🔧 后端配置优化

#### `backend-environment-config.md`
**改为嵌套类配置设计**：
```python
# 旧格式（扁平）
DB_USER=postgres
NATS_URL=nats://localhost:4222

# 新格式（嵌套）
DATABASE__USER=postgres
NATS__URL=nats://localhost:4222
```

**配置类结构**：
- `DatabaseConfig` - 数据库配置
- `NATSConfig` - 消息队列配置
- `S3Config` - 对象存储配置
- `FileStorageConfig` - 文件存储配置
- `Settings` - 主配置类（嵌套所有配置）

**参考实现**：`coding/OneManage/core/config.py`

#### `backend-architecture.md`
**数据库会话分离**：
- `database_async.py` - 异步会话（FastAPI 路由、TaskIQ 任务、事件处理器）
- `database_sync.py` - 同步会话（数据分析、Polars 处理、批量 ETL）

**使用场景**：
```python
# 异步会话（FastAPI 依赖注入）
from core.database_async import get_session

# 异步会话（上下文管理器）
from core.database_async import get_session_context

# 同步会话（数据分析）
from core.database_sync import get_sync_session_context
```

### 📝 开发流程规范更新

#### `spec-conventions.md`
**新增文件操作确认规则**：
- 明确需要用户确认的操作类型
- 定义明确指令的识别标准（"创建"、"生成"、"修改"）
- 提供标准确认流程
- 示例对比（需要确认 vs 直接执行）

**防止未经同意擅自创建或修改文件**

#### `vibe-mode.md`
**添加文件操作规则（简化版）**：
- 保持 Vibe 模式的快速迭代特点
- 同时确保文件操作的安全性
- 明确指令后立即执行，无需冗长流程

### 📚 新增辅助文档

#### `kiro-cli-tips.md`
Kiro CLI 使用技巧文档：
- 基础命令（启动、退出、帮助）
- 上下文管理（添加、查看、清除文件）
- 引用文件技巧（3 种方式）
- Steering 规则应用
- 项目开发技巧（代码审查、生成、重构）
- 常用场景示例
- 提示词技巧
- 故障排查
- 完整工作流示例

### 📊 当前 Steering 文件结构

```
steering/
├── backend-architecture.md          # 后端架构规范 ⭐ 更新
├── backend-code-style.md            # 后端代码风格
├── backend-data-processing.md       # 后端数据处理
├── backend-deployment.md            # 后端部署配置
├── backend-environment-config.md    # 后端环境配置 ⭐ 更新
├── backend-logging.md               # 后端日志规范
├── backend-tech-stack.md            # 后端技术栈
├── frontend-architecture.md         # 前端架构规范 ⭐ 新增
├── frontend-code-style.md           # 前端代码风格 ⭐ 新增
├── frontend-tech-stack.md           # 前端技术栈 ⭐ 更新
├── spec-conventions.md              # Spec 开发规范 ⭐ 更新
├── spec-task-scope.md               # Spec 任务范围
├── task-management.md               # 任务管理规范
└── vibe-mode.md                     # Vibe 模式配置 ⭐ 更新
```

### ✨ 主要改进

1. **前端规范体系完整** - 从架构到代码风格，覆盖全面
2. **配置管理现代化** - 嵌套类设计，类型安全
3. **数据库会话分离** - 异步/同步场景明确
4. **文件操作安全** - 防止误操作，需明确确认
5. **开发体验优化** - 提供完整的使用技巧文档

### 🎯 规范覆盖范围

- ✅ **后端**: 架构、代码风格、技术栈、环境配置、日志、数据处理、部署
- ✅ **前端**: 架构、代码风格、技术栈
- ✅ **开发流程**: Spec 规范、任务管理、Vibe 模式
- ✅ **工具使用**: Kiro CLI 技巧

---

**操作人员**: Kiro AI Assistant  
**操作时间**: 2025-11-30  
**操作类型**: 规范完善 + 新增文档
