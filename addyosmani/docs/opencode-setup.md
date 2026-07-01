# OpenCode 配置指南（OpenCode Setup）

本指南介绍如何在 OpenCode 中使用 Agent Skills，使其尽可能接近 Claude Code 的体验（自动技能选择、生命周期驱动的工作流和严格的流程执行）。

## 概述（Overview）

OpenCode 支持自定义 `/commands`，但没有像 Claude Code 那样的原生插件系统或自动技能路由。

因此，我们通过以下方式实现等效体验：

- 强大的系统提示词（`AGENTS.md`）
- 内置的 `skill` 工具
- 从 `/skills` 目录进行一致的技能发现

这创建了一个**智能体驱动的工作流**，技能被自动选择和执行。

虽然可以在 OpenCode 中重建 `/spec`、`/plan` 等命令，但此集成刻意采用智能体驱动的方式：

- 技能基于意图自动选择
- 工作流通过 `AGENTS.md` 强制执行
- 无需手动调用命令

这更接近 Claude Code 的实际行为，技能是自动触发而非手动触发的。

---

## 安装（Installation）

1. 克隆仓库：

```bash
git clone https://github.com/addyosmani/agent-skills.git
```

2. 在 OpenCode 中打开项目。

3. 确保工作区中存在以下文件：

- `AGENTS.md`（根目录）
- `skills/` 目录

无需额外安装。

---

## 工作原理（How It Works）

### 1. 技能发现（Skill Discovery）

所有技能位于：

```
skills/<skill-name>/SKILL.md
```

OpenCode 智能体通过 `AGENTS.md` 被指示：

- 检测何时适用某个技能
- 调用 `skill` 工具
- 严格遵循技能

### 2. 自动技能调用（Automatic Skill Invocation）

智能体评估每个请求并将其映射到相应的技能。

示例：

- "构建一个功能" → `incremental-implementation` + `test-driven-development`
- "设计一个系统" → `spec-driven-development`
- "修复一个 bug" → `debugging-and-error-recovery`
- "审查这段代码" → `code-review-and-quality`

用户**不需要**显式请求技能。

### 3. 生命周期映射（隐式命令）（Lifecycle Mapping - Implicit Commands）

开发生命周期隐式编码：

- 定义（DEFINE）→ `spec-driven-development`
- 计划（PLAN）→ `planning-and-task-breakdown`
- 构建（BUILD）→ `incremental-implementation` + `test-driven-development`
- 验证（VERIFY）→ `debugging-and-error-recovery`
- 审查（REVIEW）→ `code-review-and-quality`
- 发布（SHIP）→ `shipping-and-launch`

这取代了 `/spec`、`/plan` 等斜杠命令。

---

## 使用示例（Usage Examples）

### 示例 1：功能开发（Feature Development）

用户：
```
为这个应用添加认证功能
```

智能体行为：
- 检测到功能开发
- 调用 `spec-driven-development`
- 在编写代码前生成规格说明
- 进入计划和实现技能

---

### 示例 2：Bug 修复（Bug Fix）

用户：
```
这个端点返回 500 错误
```

智能体行为：
- 调用 `debugging-and-error-recovery`
- 复现 → 定位 → 修复 → 添加防护

---

### 示例 3：代码审查（Code Review）

用户：
```
审查这个 PR
```

智能体行为：
- 调用 `code-review-and-quality`
- 应用结构化审查（正确性、设计、可读性等）

---

## 智能体期望（关键）（Agent Expectations - Critical）

要使 OpenCode 正确工作，智能体必须遵循以下规则：

- 在行动前始终检查是否有适用的技能
- 如果技能适用，**必须**使用它
- 永不跳过必要的工作流（规格说明、计划、测试等）
- 不要直接跳到实现

这些规则通过 `AGENTS.md` 强制执行。

---

## 局限性（Limitations）

- 没有原生斜杠命令（通过意图映射处理）
- 没有插件系统（通过提示词 + 结构处理）
- 技能调用依赖于模型合规性

尽管如此，该工作流在实践中非常接近 Claude Code。

---

## 推荐工作流（Recommended Workflow）

只需使用自然语言：

- "设计一个功能"
- "计划这次变更"
- "实现这个"
- "修复这个 bug"
- "审查这个"

智能体会自动选择并执行正确的技能。

---

## 总结（Summary）

OpenCode 集成通过以下方式工作：

- 结构化技能（本仓库）
- 强大的智能体规则（`AGENTS.md`）
- 通过推理自动调用技能

这实现了**完全智能体驱动的、生产级工程工作流**，无需插件或手动命令。
