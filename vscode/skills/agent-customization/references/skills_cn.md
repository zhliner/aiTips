# [Agent Skills（SKILL.md）](https://code.visualstudio.com/docs/copilot/customization/agent-skills)

由指令、脚本和资源组成的文件夹，agents 在需要时按需加载以执行专门任务。

## 结构

```
.github/skills/<skill-name>/
├── SKILL.md           # 必需（名称必须与文件夹匹配）
├── scripts/           # 可执行代码
├── references/        # 按需加载的文档
└── assets/            # 模板、样板文件
```

## 位置

| 路径 | 作用域 |
|------|--------|
| `.github/skills/<name>/` | 项目 |
| `.agents/skills/<name>/` | 项目 |
| `.claude/skills/<name>/` | 项目 |
| `~/.copilot/skills/<name>/` | 个人 |
| `~/.agents/skills/<name>/` | 个人 |
| `~/.claude/skills/<name>/` | 个人 |

## SKILL.md 格式

```yaml
---
name: skill-name              # 必填：1-64 个字符，小写字母数字加连字符，必须与文件夹匹配
description: '何时以及为何使用。最多 1024 个字符。'
argument-hint: '斜杠调用时显示的可选提示'
user-invocable: true          # 可选：作为斜杠命令显示（默认：true）
disable-model-invocation: false # 可选：禁用模型自动触发的加载
---
```

### 正文

- 该 skill 完成的功能
- 何时使用（触发条件和使用场景）
- 逐步操作流程
- 资源引用：`[script](./scripts/test.js)`

## 模板

```markdown
---
name: webapp-testing
description: 'Test web applications using Playwright. Use for verifying frontend, debugging UI, capturing screenshots.'
---

# Web Application Testing

## When to Use
- Verify frontend functionality
- Debug UI behavior

## Procedure
1. Start the web server
2. Run [test script](./scripts/test.js)
3. Review screenshots in `./screenshots/`
```

## 渐进式加载

1. **发现**（约 100 tokens）：Agent 读取 `name` 和 `description`
2. **指令**（少于 5000 tokens）：在相关时加载 `SKILL.md` 正文
3. **资源**：仅在引用时加载额外文件

保持文件引用从 `SKILL.md` 出发只有一层深度。

## 斜杠命令行为

Skills 和 prompt 文件在聊天中输入 `/` 后都会出现。

| 配置 | 斜杠命令 | 自动加载 |
|------|----------|----------|
| 默认（两者均省略） | 是 | 是 |
| `user-invocable: false` | 否 | 是 |
| `disable-model-invocation: true` | 是 | 否 |
| 两者均设置 | 否 | 否 |

## 适用场景

可重复的、按需的工作流，带有捆绑资源（脚本、模板、参考文档）。

## 核心原则

1. **关键词丰富的 description**：包含用于发现的触发词
2. **渐进式加载**：保持 SKILL.md 在 500 行以内；使用引用文件
3. **相对路径**：skill 资源始终使用 `./`
4. **自包含**：包含完成任务所需的所有过程性知识

## 反模式

- **模糊的 description**："A helpful skill" 无法实现发现
- **庞大的 SKILL.md**：所有内容堆在一个文件中而非使用引用文件
- **名称不匹配**：文件夹名称与 `name` 字段不一致
- **缺少流程说明**：只有描述而没有逐步指导
