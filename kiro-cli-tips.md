# Kiro CLI 使用技巧

## 基础命令

### 启动对话
```bash
kiro-cli chat
```

### 退出对话
```
/quit
```

### 查看帮助
```bash
kiro-cli --help
```

## 上下文管理

### 添加文件到上下文
```
/context add /path/to/file.md
/context add steering/frontend-code-style.md
```

### 添加多个文件
```
/context add steering/*.md
```

### 查看当前上下文
```
/context list
```

### 清除上下文
```
/context clear
```

## 引用文件技巧

### 方式 1：直接在对话中提及
```
请查看 steering/frontend-code-style.md 并检查我的代码
```

### 方式 2：使用绝对路径
```
请根据 /home/chris/coding/.kiro/steering/frontend-architecture.md 重构组件
```

### 方式 3：相对路径
```
查看 ../OneManage_web/src/components/TaskList.vue
```

## Steering 规则应用

### 引用前端规范
```
请按照 steering/frontend-code-style.md 的规范重构这段代码
```

### 引用后端规范
```
请按照 steering/backend-code-style.md 检查这个 API
```

### 引用架构规范
```
根据 steering/frontend-architecture.md 设计这个功能
```

## 项目开发技巧

### 1. 代码审查
```
请根据 steering/frontend-code-style.md 审查以下代码：
[粘贴代码]
```

### 2. 生成符合规范的代码
```
请按照 steering/frontend-code-style.md 创建一个任务列表组件
```

### 3. 重构现有代码
```
请根据 steering/frontend-architecture.md 重构 src/views/TaskList.vue
```

### 4. 检查规范遵循情况
```
检查 OneManage_web/src/components/ 下的组件是否符合 steering 规范
```

## 多文件操作

### 批量检查
```
检查 OneManage_web/src/api/ 下所有文件是否符合 steering/frontend-code-style.md
```

### 对比文件
```
对比 steering/frontend-code-style.md 和 steering/backend-code-style.md 的异同
```

## 环境变量

### 设置 API 端点
```bash
export KIRO_API_ENDPOINT=https://your-endpoint
```

### 设置配置文件路径
```bash
export KIRO_CONFIG_PATH=/path/to/config
```

## 最佳实践

### 1. 项目初始化时
```
请根据 steering/ 目录下的所有规范，帮我搭建项目结构
```

### 2. 开发新功能前
```
请根据 specs/xxx/requirements.md 和 steering 规范，设计实现方案
```

### 3. 代码提交前
```
请根据 steering/frontend-code-style.md 的检查清单审查我的代码
```

### 4. 学习规范
```
总结 steering/frontend-architecture.md 的核心要点
```

## 常用场景

### 场景 1：创建新组件
```
请按照 steering/frontend-code-style.md 和 steering/frontend-architecture.md
创建一个任务卡片组件，包含以下功能：
- 显示任务名称和状态
- 支持删除操作
- 使用 Naive UI
```

### 场景 2：API 封装
```
请按照 steering/frontend-code-style.md 的 API 调用规范
封装用户相关的 API 接口
```

### 场景 3：状态管理
```
请按照 steering/frontend-architecture.md 的状态管理策略
创建一个任务管理的 Pinia Store
```

### 场景 4：代码重构
```
请根据 steering/frontend-code-style.md 重构以下代码，
重点关注：
- 函数命名
- 错误处理
- 注释规范
[粘贴代码]
```

## 提示词技巧

### 明确引用规范
```
✅ 请按照 steering/frontend-code-style.md 的"函数规范"部分
❌ 请按照规范
```

### 指定检查项
```
✅ 检查代码是否符合 steering/frontend-code-style.md 的：
   - 函数命名规范
   - JSDoc 注释
   - 错误处理
❌ 检查代码
```

### 提供上下文
```
✅ 我在开发 OneManage_web 项目，请根据 steering/frontend-tech-stack.md
   配置 Vite
❌ 帮我配置 Vite
```

## 快捷操作

### 快速审查
```
/context add steering/frontend-code-style.md
审查以下代码
```

### 快速生成
```
/context add steering/frontend-architecture.md
创建一个符合规范的用户管理页面
```

## 注意事项

1. **路径准确性**：确保文件路径正确，使用 Tab 补全
2. **规范版本**：确保引用的规范文件是最新版本
3. **上下文大小**：避免一次性添加过多文件到上下文
4. **清晰表达**：明确说明要遵循哪个规范的哪个部分

## 高级技巧

### 1. 组合多个规范
```
请同时参考：
- steering/frontend-code-style.md（代码风格）
- steering/frontend-architecture.md（架构设计）
- steering/frontend-tech-stack.md（技术选型）
创建一个完整的数据导入功能
```

### 2. 规范对比
```
对比 steering/frontend-code-style.md 和 steering/backend-code-style.md
总结前后端代码规范的共同点和差异
```

### 3. 规范演进
```
根据项目实践，建议 steering/frontend-code-style.md 需要补充哪些内容
```

## 故障排查

### 文件找不到
```bash
# 检查文件是否存在
ls -la /home/chris/coding/.kiro/steering/

# 使用绝对路径
/context add /home/chris/coding/.kiro/steering/frontend-code-style.md
```

### 上下文过大
```
# 清除上下文
/context clear

# 只添加需要的文件
/context add steering/frontend-code-style.md
```

## 工作流示例

### 完整开发流程
```bash
# 1. 启动 kiro-cli
kiro-cli chat

# 2. 添加规范到上下文
/context add steering/frontend-code-style.md
/context add steering/frontend-architecture.md

# 3. 查看需求
请查看 specs/xxx/requirements.md 并设计实现方案

# 4. 生成代码
请创建符合规范的组件

# 5. 代码审查
请审查代码是否符合规范

# 6. 退出
/quit
```
