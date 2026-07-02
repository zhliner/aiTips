# Definition of Done（完成标准）

一个项目级的标准门槛，每一项变更都必须满足才能算作"完成"。与 Acceptance Criteria（验收标准）不同——后者因任务而异，回答的是"我们做对了吗？"；而 Definition of Done 每次都一样，回答的是"这符合我们的标准吗？"。将其作为 `planning-and-task-breakdown`、`incremental-implementation` 和 `shipping-and-launch` 中的最终关卡。

## Definition of Done 与 Acceptance Criteria 的区别

| | Acceptance Criteria（验收标准） | Definition of Done（完成标准） |
|---|---|---|
| 范围 | 特定于某一任务或规格 | 适用于每一次增量 |
| 变化 | 每项内容各不相同 | 固定不变，可复用 |
| 回答 | "我们做好*这件事*了吗？" | "它*准备好*了吗？" |
| 负责人 | 在规划任务时定义 | 为项目一次性定义 |
| 示例 | "用户可以通过邮件链接重置密码" | "测试通过，无回归问题，文档已更新" |

两者互为补充。只有当**其** Acceptance Criteria 已满足**且**项目级 Definition of Done 也已满足时，一项任务才算完成。跳过任何一项，都会导致有工作看起来完成但实际上并未完成。

## 标准检查清单

在宣布每一项变更完成之前，逐一检查。

### Correctness（正确性）
- [ ] 任务的所有 Acceptance Criteria 均已满足
- [ ] 代码在运行时能按预期运行并表现正确，而非仅仅编译通过或类型检查通过
- [ ] 新行为有测试覆盖，且这些测试在变更前失败、变更后通过
- [ ] 已有测试仍然通过；未引入回归问题
- [ ] 边界情况和错误路径均已被处理，而不仅仅是正常路径

### Quality（质量）
- [ ] 代码通过命名和结构表达意图；无需注释来解释其*做了什么*
- [ ] 无重复的业务逻辑
- [ ] 无死代码、调试输出或被注释掉的代码块残留
- [ ] 变更范围仅限于该任务；没有混入无关的重构
- [ ] Lint 和格式化检查通过

这些条目背后的深度内容参见 `code-review-and-quality`（五维代码审查）和 `code-simplification`（在不改变行为的前提下降低复杂度）。

### Integration（集成）
- [ ] 变更能与系统其余部分协同工作，而不仅仅是孤立运行
- [ ] 数据库迁移、配置变更和 feature flag 均已考虑在内
- [ ] 对于任何公共接口或 API 变更，已考虑向后兼容性

### Documentation（文档）
- [ ] 公共接口、API 和面向用户的行为均已文档化
- [ ] 值得保留的架构决策已被记录（参见 `documentation-and-adrs`）
- [ ] 文档使用无时间性的语言描述当前状态，而非变更历史

### Ship-readiness（发布就绪）
- [ ] 对于任何不受信任的输入、身份验证或数据处理，已审查安全影响（参见 `security-and-hardening`）
- [ ] 对于新的关键路径，可观测性已就位（日志、指标、追踪）（参见 `observability-and-instrumentation`）
- [ ] 对于任何有风险的变更，存在回滚路径（参见 `shipping-and-launch`）
- [ ] 人类已在合并或部署前评审并批准

## 如何应用

- **每个任务**：在打勾完成任务前，确认 Correctness 和 Quality 部分。
- **每个功能**：在认为功能完成前，确认 Integration 和 Documentation 部分。
- **每个发布版本**：完整检查清单是底线；`shipping-and-launch` 在此基础上增加部署特定的关卡。

一次性根据项目定制此清单，然后不加修改地复用。一个每个 Sprint 都要重新协商的 Definition of Done，不是真正的 Definition of Done。

## Red Flags（警示信号）

- "做完了，只是还没运行过"：未经验证的工作不算完成。
- 将"测试通过"当作完成的同义词，而跳过文档、回归测试或运行时验证。
- 因截止时间压力而降低标准。
- 将 Acceptance Criteria 视为完整标准，没有项目级质量底线。
- 在需要人类评审的变更上，未经评审就宣称"完成"。
