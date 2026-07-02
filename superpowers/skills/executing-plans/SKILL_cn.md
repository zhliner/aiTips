---
name: executing-plans
description: 当你有书面实施计划需要在单独的会话中执行并设置审查检查点时使用
---

# 执行计划（Executing Plans）

## 概述

加载计划、批判性审查、执行所有任务、完成后报告。

**开始时宣布：** "I'm using the executing-plans skill to implement this plan."

**注意：** 告诉你的用户搭档，Superpowers 在可以使用 subagent 的情况下效果更好。如果在支持 subagent 的平台上运行，其工作质量将显著提高（Claude Code、Codex CLI、Codex App 和 Copilot CLI 都符合条件；参见 `../using-superpowers/references/` 中的各平台工具参考）。如果 subagent 可用，请使用 superpowers:subagent-driven-development 而非此 skill。

## 流程

### 步骤 1：加载并审查计划
1. 读取计划文件
2. 批判性审查——识别计划中的任何疑问或问题
3. 如有疑虑：在开始前向用户搭档提出
4. 如无疑虑：为计划项创建 todos 并继续

### 步骤 2：执行任务

对每个任务：
1. 标记为 in_progress
2. 严格按照每个步骤执行（计划包含细粒度步骤）
3. 按规定运行验证
4. 标记为已完成

### 步骤 3：完成开发

所有任务完成并验证后：
- 宣布："I'm using the finishing-a-development-branch skill to complete this work."
- **必需子 skill：** 使用 superpowers:finishing-a-development-branch
- 按照该 skill 验证测试、展示选项、执行选择

## 何时停下来寻求帮助

**在以下情况立即停止执行：**
- 遇到阻碍（缺少依赖、测试失败、指令不明确）
- 计划存在关键缺口导致无法开始
- 你不理解某条指令
- 验证反复失败

**宁可请求澄清也不要猜测。**

## 何时重新审视早期步骤

**在以下情况返回审查（步骤 1）：**
- 搭档根据你的反馈更新了计划
- 基本方法需要重新思考

**不要强行通过阻碍**——停下来询问。

## 记住
- 先批判性审查计划
- 严格按照计划步骤执行
- 不要跳过验证
- 计划要求时引用相关 skills
- 遇到阻碍时停下，不要猜测
- 未经用户明确同意，不要在 main/master 分支上开始实现

## 集成

**必需的工作流 skills：**
- **superpowers:using-git-worktrees**——确保隔离的工作区（创建一个或验证已有的）
- **superpowers:writing-plans**——创建此 skill 执行的计划
- **superpowers:finishing-a-development-branch**——所有任务完成后完成开发
