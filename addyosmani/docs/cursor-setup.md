# 在 Cursor 中使用 agent-skills

## 安装配置（Setup）

### 方式一：Rules 目录（推荐）

Cursor 支持 `.cursor/rules/` 目录用于项目特定规则：

```bash
# 创建 rules 目录
mkdir -p .cursor/rules

# 复制你想作为规则使用的技能
cp /path/to/agent-skills/skills/test-driven-development/SKILL.md .cursor/rules/test-driven-development.md
cp /path/to/agent-skills/skills/code-review-and-quality/SKILL.md .cursor/rules/code-review-and-quality.md
cp /path/to/agent-skills/skills/incremental-implementation/SKILL.md .cursor/rules/incremental-implementation.md
```

该目录中的规则会自动加载到 Cursor 的上下文中。

### 方式二：.cursorrules 文件

在项目根目录创建 `.cursorrules` 文件，内联核心技能：

```bash
# 生成合并的规则文件
cat /path/to/agent-skills/skills/test-driven-development/SKILL.md > .cursorrules
echo "\n---\n" >> .cursorrules
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md >> .cursorrules
```

## 推荐配置（Recommended Configuration）

### 核心技能（始终加载）（Essential Skills - Always Load）

将这些添加到 `.cursor/rules/`：

1. `test-driven-development.md` — TDD 工作流和 Prove-It 模式
2. `code-review-and-quality.md` — 五维度审查
3. `incremental-implementation.md` — 以小型可验证的切片构建

### 阶段特定技能（按需加载）（Phase-Specific Skills - Load on Demand）

对于阶段特定的工作，按需创建额外的规则文件：

- `spec-development.md` -> `spec-driven-development/SKILL.md`
- `frontend-ui.md` -> `frontend-ui-engineering/SKILL.md`
- `security.md` -> `security-and-hardening/SKILL.md`
- `performance.md` -> `performance-optimization/SKILL.md`

在处理相关任务时将这些添加到 `.cursor/rules/`，完成后移除以管理上下文限制。

## 使用技巧（Usage Tips）

1. **不要一次加载所有技能** - Cursor 有上下文限制。加载 2-3 个核心技能作为规则，按需添加阶段特定技能。
2. **显式引用技能** - 告诉 Cursor "按照 test-driven-development 规则进行这次修改"，以确保它读取已加载的规则。
3. **使用智能体进行审查** - 复制 `agents/code-reviewer.md` 的内容，告诉 Cursor "使用这个代码审查框架审查这个 diff。"
4. **按需加载参考** - 处理性能问题时，将 `performance.md` 添加到 `.cursor/rules/` 或直接粘贴清单内容。
