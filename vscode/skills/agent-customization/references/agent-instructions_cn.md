# [agent instructions（全局指令）](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)

自动应用于整个 workspace 中所有聊天请求的指南。

## 文件类型（二选一）

| 文件 | 位置 | 用途 |
|------|------|------|
| `copilot-instructions.md` | `.github/` | 项目级规范（推荐，跨编辑器） |
| `AGENTS.md` | 根目录或子文件夹 | 开放标准，支持 monorepo 层级 |

**只选其一**——不要同时使用两者。

## 模板

仅包含 workspace 受益的部分：

```markdown
# Project Guidelines

## Code Style
{语言和格式偏好——引用体现模式的关键文件}

## Architecture
{主要组件、服务边界、结构决策背后的"原因"}

## Build and Test
{安装、构建、测试的命令——agents 会尝试运行这些命令}

## Conventions
{与常见实践不同的模式——包含具体示例}
```

对于大型仓库，链接到详细文档而非内嵌内容：`See docs/TESTING.md for test conventions.`

## 适用场景

- 普遍适用于所有场景的通用编码规范
- 通过版本控制共享的团队偏好
- 项目级要求（测试、文档）

## 核心原则

1. **默认最小化**：仅包含与*每个*任务相关的内容
2. **简洁且可操作**：每一行都应指导行为
3. **链接而非内嵌**：引用文档而非复制内容。搜索现有文档（`docs/**/*.md`、`CONTRIBUTING.md` 等）并梳理其覆盖范围——仅内联其他文档未记录的对 agent 至关重要的注意事项
4. **保持更新**：实践变化时同步更新

## 反模式

- **同时使用两种文件类型**：同时拥有 `copilot-instructions.md` 和 `AGENTS.md`
- **大杂烩**：包含所有内容而非最重要的内容
- **重复文档**：复制 README 而非链接
- **显而易见的指令**：已被 linter 强制执行的规范
