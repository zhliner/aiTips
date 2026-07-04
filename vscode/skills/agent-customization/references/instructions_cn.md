# [文件级指令（.instructions.md）](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)

在当前任务相关时按需加载的指南，或在文件匹配模式时显式加载。

## 位置

| 路径 | 作用域 |
|------|--------|
| `.github/instructions/*.instructions.md` | Workspace |
| `<profile>/instructions/*.instructions.md` | 用户配置 |

## Frontmatter

```yaml
---
description: "<必填>"    # 用于按需发现——包含丰富关键词
name: "Instruction Name"     # 可选，默认为文件名
applyTo: "**/*.ts"           # 可选，匹配文件时自动附加
---
```

## 发现模式

| 模式 | 触发条件 | 使用场景 |
|------|----------|----------|
| **按需**（`description`） | Agent 检测到任务相关性 | 基于任务：迁移、重构、API 工作 |
| **显式**（`applyTo`） | 上下文中的文件匹配 glob | 基于文件：语言规范、框架规则 |
| **手动** | `Add Context` → `Instructions` | 临时附加 |

## 模板

```markdown
---
description: "Use when writing database migrations, schema changes, or data transformations. Covers safety checks and rollback patterns."
---
# Migration Guidelines

- Always create reversible migrations
- Test rollback before merging
- Never drop columns in the same release as code removal
```

注意 description 中的 "Use when..." 模式——这有助于按需发现。

## 显式文件匹配（可选）

当指令适用于特定文件类型或文件夹时，使用 `applyTo`：

```yaml
applyTo: "**"                           # 始终包含，无论文件或描述（谨慎使用）
applyTo: "**/*.py"                      # 所有 Python 文件
applyTo: ["src/**", "lib/**"]           # 多个模式（OR）
applyTo: src/**, lib/**                 # 不使用数组语法的多个模式（OR）
applyTo: "src/api/**/*.ts"              # 特定文件夹 + 扩展名
```

在创建或修改匹配文件时应用，不适用于只读操作。

## 核心原则

1. **关键词丰富的 description**：包含用于按需发现的触发词
2. **每个文件一个关注点**：测试、样式、文档分别使用不同文件
3. **简洁且可操作**：共享上下文窗口——保持聚焦
4. **展示而非叙述**：简短的代码示例优于冗长的解释

## 反模式

- **模糊的 description**："Helpful coding tips" 无法实现发现
- **过于宽泛的 applyTo**：`"**"` 配合仅与特定文件相关的内容
- **重复文档**：复制 README 而非链接
- **混合关注点**：测试 + API 设计 + 样式放在一个文件中
