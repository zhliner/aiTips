# 在 Windsurf 中使用 agent-skills

## 安装配置（Setup）

### 项目规则（Project Rules）

Windsurf 使用 `.windsurfrules` 存放项目特定的智能体指令：

```bash
# 从最重要的技能创建合并的规则文件
cat /path/to/agent-skills/skills/test-driven-development/SKILL.md > .windsurfrules
echo "\n---\n" >> .windsurfrules
cat /path/to/agent-skills/skills/incremental-implementation/SKILL.md >> .windsurfrules
echo "\n---\n" >> .windsurfrules
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md >> .windsurfrules
```

### 全局规则（Global Rules）

对于你想在所有项目中使用的技能，将它们添加到 Windsurf 的全局规则：

1. 打开 Windsurf → 设置 → AI → 全局规则
2. 粘贴你最常用的技能内容

## 推荐配置（Recommended Configuration）

保持 `.windsurfrules` 聚焦于 2-3 个核心技能，以控制在上下文限制内：

```
# .windsurfrules
# 本项目的核心 agent-skills

[粘贴 test-driven-development SKILL.md]

---

[粘贴 incremental-implementation SKILL.md]

---

[粘贴 code-review-and-quality SKILL.md]
```

## 使用技巧（Usage Tips）

1. **有选择地加载** — Windsurf 的上下文有限。选择能解决你最大质量缺口的技能。
2. **在对话中引用** — 在处理特定阶段时，将额外的技能内容粘贴到聊天中（例如，在构建认证时粘贴 `security-and-hardening`）。
3. **将参考资料作为清单** — 粘贴 `references/security-checklist.md` 并要求 Windsurf 验证每一项。
