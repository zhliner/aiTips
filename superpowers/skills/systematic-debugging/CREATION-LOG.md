# Creation Log: Systematic Debugging Skill（创建日志：系统调试技能）

提取、结构和防弹关键技能的参考示例。

## Source Material（源材料）

从 `/Users/jesse/.claude/CLAUDE.md` 提取的调试框架：
- 4 阶段系统过程（调查 → 模式分析 → 假设 → 实现）
- 核心授权：总是找到根因，永远不要修复症状
- 规则设计为抵抗时间压力和合理化

## Extraction Decisions（提取决策）

**包含什么：**
- 完整的 4 阶段框架及所有规则
- 反捷径（"永远不要修复症状"、"停止并重新分析"）
- 抗压语言（"即使更快"、"即使我看起来很着急"）
- 每个阶段的具体步骤

**排除什么：**
- 项目特定上下文
- 相同规则的重复变体
- 叙述性解释（浓缩为原则）

## Structure Following skill-creation/SKILL.md（遵循 skill-creation/SKILL.md 的结构）

1. **丰富的 when_to_use** - 包括症状和反模式
2. **类型：技术** - 带步骤的具体过程
3. **关键词** - "root cause"、"symptom"、"workaround"、"debugging"、"investigation"
4. **流程图** - "修复失败"的决策点 → 重新分析与添加更多修复
5. **逐阶段分解** - 可扫描的清单格式
6. **反模式部分** - 不该做什么（对此技能至关重要）

## Bulletproofing Elements（防弹元素）

框架设计为在压力下抵抗合理化：

### Language Choices（语言选择）
- "ALWAYS" / "NEVER"（不是"should" / "try to"）
- "even if faster" / "even if I seem in a hurry"
- "STOP and re-analyze"（明确暂停）
- "Don't skip past"（捕获实际行为）

### Structural Defenses（结构防御）
- **阶段 1 必需** - 不能跳到实现
- **单一假设规则** - 强制思考，防止散弹枪修复
- **显式失败模式** - "IF your first fix doesn't work" 带强制行动
- **反模式部分** - 准确显示捷径的样子

### Redundancy（冗余）
- 根因授权在 overview + when_to_use + 阶段 1 + 实现规则中
- "永远不要修复症状"在不同上下文中出现 4 次
- 每个阶段都有明确的"不要跳过"指导

## Testing Approach（测试方法）

创建了 4 个验证测试，遵循 skills/meta/testing-skills-with-subagents：

### Test 1: Academic Context (No Pressure)（测试 1：学术上下文（无压力））
- 简单错误，无时间压力
- **结果：** 完美合规，完整调查

### Test 2: Time Pressure + Obvious Quick Fix（测试 2：时间压力 + 明显的快速修复）
- 用户"很着急"，症状修复看起来容易
- **结果：** 抵抗捷径，遵循完整过程，找到真正的根因

### Test 3: Complex System + Uncertainty（测试 3：复杂系统 + 不确定性）
- 多层失败，不清楚是否能找到根因
- **结果：** 系统调查，追踪所有层，找到源头

### Test 4: Failed First Fix（测试 4：第一次修复失败）
- 假设不起作用，诱惑添加更多修复
- **结果：** 停止，重新分析，形成新假设（没有散弹枪）

**所有测试通过。** 没有发现合理化。

## Iterations（迭代）

### Initial Version（初始版本）
- 完整的 4 阶段框架
- 反模式部分
- "修复失败"决策流程图

### Enhancement 1: TDD Reference（增强 1：TDD 引用）
- 添加了指向 skills/testing/test-driven-development 的链接
- 说明 TDD 的"最简单代码" ≠ 调试的"根因"
- 防止方法论之间的混淆

## Final Outcome（最终结果）

防弹技能：
- ✅ 清楚地授权根因调查
- ✅ 抵抗时间压力合理化
- ✅ 为每个阶段提供具体步骤
- ✅ 明确显示反模式
- ✅ 在多个压力场景下测试
- ✅ 澄清与 TDD 的关系
- ✅ 准备使用

## Key Insight（关键见解）

**最重要的防弹：** 反模式部分显示在当下感觉合理的准确捷径。当 Claude 想"我就添加这一个快速修复"时，看到列为错误的准确模式会产生认知摩擦。

## Usage Example（使用示例）

遇到错误时：
1. 加载技能：skills/debugging/systematic-debugging
2. 阅读概述（10 秒） - 提醒授权
3. 遵循阶段 1 清单 - 强制调查
4. 如果想跳过 - 看到反模式，停止
5. 完成所有阶段 - 找到根因

**时间投资：** 5-10 分钟
**节省时间：** 数小时的打地鼠式修复症状

---

*Created: 2025-10-03*
*Purpose: Reference example for skill extraction and bulletproofing*
