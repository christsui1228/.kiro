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

**操作人员**: Kiro AI Assistant  
**操作时间**: 2025-10-20  
**操作类型**: 文件重命名 + 配置更新


---

## 📅 2025-10-20 (续)

### 🔄 Spec 目录重命名

为了更清晰地表明 spec 的范围，对 specs 目录进行了重命名：

| 旧名称 | 新名称 | 说明 |
|-------|-------|------|
| `psd-layer-preview/` | `fullstack-psd-layer-preview/` | 明确这是一个全栈功能（主要后端 + 前端验证） |

### 📝 命名规范

Spec 目录命名规范：
- `backend-*` - 纯后端功能
- `frontend-*` - 纯前端功能
- `fullstack-*` - 前后端协同功能
- `infra-*` - 基础设施相关

### ✅ 当前 Specs 结构

```
.kiro/specs/
├── fullstack-psd-layer-preview/     ← 重命名
│   ├── requirements.md
│   ├── design.md
│   └── tasks.md
└── unified-taskiq-nats-architecture/
    └── ...
```
