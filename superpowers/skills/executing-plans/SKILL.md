---
name: executing-plans
description: 当你有一个书面的实施计划需要在独立会话中执行并带有审查检查点时使用
---

# 执行计划（Executing Plans）

## 概述

加载计划，批判性审查，执行所有任务，完成后报告。

**开始时宣布：** "我正在使用 executing-plans 技能来实施此计划。"

**注意：** 告诉你的合作者，Superpowers 在有子代理访问权限时效果更佳。如果在支持子代理的平台上运行，其工作质量会显著更高（Claude Code、Codex CLI、Codex App 和 Copilot CLI 都符合条件；参见 `../using-superpowers/references/` 中的各平台工具参考）。如果子代理可用，请使用 superpowers:subagent-driven-development 替代此技能。

## 流程

### 步骤 1：加载并审查计划
1. 读取计划文件
2. 批判性审查——识别关于计划的任何问题或疑虑
3. 如有疑虑：在开始之前向你的合作者提出
4. 如无疑虑：为计划项目创建待办事项并继续

### 步骤 2：执行任务

对于每个任务：
1. 标记为 in_progress
2. 严格遵循每个步骤（计划中有细粒度的步骤）
3. 按照规定运行验证
4. 标记为已完成

### 步骤 3：完成开发

所有任务完成并验证后：
- 宣布："我正在使用 finishing-a-development-branch 技能来完成此工作。"
- **必需的子技能：** 使用 superpowers:finishing-a-development-branch
- 遵循该技能验证测试、呈现选项、执行选择

## 何时停止并寻求帮助

**在以下情况立即停止执行：**
- 遇到阻塞因素（缺少依赖、测试失败、指令不清楚）
- 计划存在阻碍开始的关键缺失
- 你不理解某个指令
- 验证反复失败

**请求澄清而非猜测。**

## 何时回访之前的步骤

**在以下情况返回审查（步骤 1）：**
- 合作者根据你的反馈更新了计划
- 基本方法需要重新思考

**不要强行通过阻塞因素**——停下来询问。

## 记住
- 首先批判性审查计划
- 严格遵循计划步骤
- 不要跳过验证
- 当计划指示时引用技能
- 阻塞时停止，不要猜测
- 绝不在没有明确用户同意的情况下在 main/master 分支上开始实施

## 集成

**必需的工作流技能：**
- **superpowers:using-git-worktrees** - 确保隔离的工作空间（创建一个或验证现有）
- **superpowers:writing-plans** - 创建此技能执行的计划
- **superpowers:finishing-a-development-branch** - 所有任务完成后完成开发
