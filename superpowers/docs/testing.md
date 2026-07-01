# Testing Superpowers Skills（测试 Superpowers 技能）

本文档描述如何测试 Superpowers 技能，特别是针对 `subagent-driven-development` 等复杂技能的集成测试。

## Overview（概述）

测试涉及子代理、工作流和复杂交互的技能，需要在无头模式下运行实际的 Claude Code 会话，并通过会话记录验证其行为。

## Test Structure（测试结构）

```
tests/
├── claude-code/
│   ├── test-helpers.sh                    # 共享测试工具
│   ├── test-subagent-driven-development-integration.sh
│   ├── analyze-token-usage.py             # Token 分析工具
│   └── run-skill-tests.sh                 # 测试运行器（如果存在）
```

## Running Tests（运行测试）

### Integration Tests（集成测试）

集成测试使用实际技能执行真实的 Claude Code 会话：

```bash
# 运行 subagent-driven-development 集成测试
cd tests/claude-code
./test-subagent-driven-development-integration.sh
```

**注意：** 集成测试可能需要 10-30 分钟，因为它们使用多个子代理执行实际的实现计划。

### Requirements（要求）

- 必须从 **superpowers 插件目录**运行（不能从临时目录）
- Claude Code 必须已安装并可通过 `claude` 命令使用
- 本地开发市场必须启用：在 `~/.claude/settings.json` 中设置 `"superpowers@superpowers-dev": true`

## Integration Test: subagent-driven-development（集成测试：subagent-driven-development）

### What It Tests（测试内容）

集成测试验证 `subagent-driven-development` 技能是否正确：

1. **Plan Loading（计划加载）**：在开始时读取一次计划
2. **Full Task Text（完整任务文本）**：向子代理提供完整的任务描述（不让它们读文件）
3. **Self-Review（自我审查）**：确保子代理在报告前进行自我审查
4. **Review Order（审查顺序）**：先运行规格合规性审查，再运行代码质量审查
5. **Review Loops（审查循环）**：发现问题时使用审查循环
6. **Independent Verification（独立验证）**：规格审查者独立阅读代码，不信任实现者的报告

### How It Works（工作方式）

1. **Setup（设置）**：创建一个带最小实现计划的临时 Node.js 项目
2. **Execution（执行）**：使用该技能在无头模式下运行 Claude Code
3. **Verification（验证）**：解析会话记录（`.jsonl` 文件）以验证：
   - Skill 工具被调用
   - 子代理被分派（Task 工具）
   - TodoWrite 被用于跟踪
   - 实现文件被创建
   - 测试通过
   - Git 提交显示正确的工作流
4. **Token Analysis（Token 分析）**：显示按子代理分类的 token 使用情况

### Test Output（测试输出）

```
========================================
 Integration Test: subagent-driven-development
========================================

Test project: /tmp/tmp.xyz123

=== Verification Tests ===

Test 1: Skill tool invoked...
  [PASS] subagent-driven-development skill was invoked

Test 2: Subagents dispatched...
  [PASS] 7 subagents dispatched

Test 3: Task tracking...
  [PASS] TodoWrite used 5 time(s)

Test 6: Implementation verification...
  [PASS] src/math.js created
  [PASS] add function exists
  [PASS] multiply function exists
  [PASS] test/math.test.js created
  [PASS] Tests pass

Test 7: Git commit history...
  [PASS] Multiple commits created (3 total)

Test 8: No extra features added...
  [PASS] No extra features added

=========================================
 Token Usage Analysis
=========================================

Usage Breakdown:
----------------------------------------------------------------------------------------------------
Agent           Description                          Msgs      Input     Output      Cache     Cost
----------------------------------------------------------------------------------------------------
main            Main session (coordinator)             34         27      3,996  1,213,703 $   4.09
3380c209        implementing Task 1: Create Add Function     1          2        787     24,989 $   0.09
34b00fde        implementing Task 2: Create Multiply Function     1          4        644     25,114 $   0.09
3801a732        reviewing whether an implementation matches...   1          5        703     25,742 $   0.09
4c142934        doing a final code review...                    1          6        854     25,319 $   0.09
5f017a42        a code reviewer. Review Task 2...               1          6        504     22,949 $   0.08
a6b7fbe4        a code reviewer. Review Task 1...               1          6        515     22,534 $   0.08
f15837c0        reviewing whether an implementation matches...   1          6        416     22,485 $   0.07
----------------------------------------------------------------------------------------------------

TOTALS:
  Total messages:         41
  Input tokens:           62
  Output tokens:          8,419
  Cache creation tokens:  132,742
  Cache read tokens:      1,382,835

  Total input (incl cache): 1,515,639
  Total tokens:             1,524,058

  Estimated cost: $4.67
  (at $3/$15 per M tokens for input/output)

========================================
 Test Summary
========================================

STATUS: PASSED
```

## Token Analysis Tool（Token 分析工具）

### Usage（用法）

分析任何 Claude Code 会话的 token 使用情况：

```bash
python3 tests/claude-code/analyze-token-usage.py ~/.claude/projects/<project-dir>/<session-id>.jsonl
```

### Finding Session Files（查找会话文件）

会话记录存储在 `~/.claude/projects/` 中，工作目录路径被编码：

```bash
# 示例：/Users/jesse/Documents/GitHub/superpowers/superpowers
SESSION_DIR="$HOME/.claude/projects/-Users-jesse-Documents-GitHub-superpowers-superpowers"

# 查找最近的会话
ls -lt "$SESSION_DIR"/*.jsonl | head -5
```

### What It Shows（显示内容）

- **Main session usage（主会话使用量）**：协调者（你或主 Claude 实例）的 token 使用量
- **Per-subagent breakdown（每个子代理明细）**：每次 Task 调用包含：
  - Agent ID
  - Description（从提示中提取）
  - Message count（消息数）
  - Input/output tokens
  - Cache usage（缓存使用量）
  - Estimated cost（预估成本）
- **Totals（总计）**：总体 token 使用量和成本估算

### Understanding the Output（理解输出）

- **High cache reads（高缓存读取）**：好——表示提示缓存正在工作
- **High input tokens on main（主会话高输入 token）**：预期行为——协调者拥有完整上下文
- **Similar costs per subagent（每个子代理成本相近）**：预期行为——每个任务复杂度相似
- **Cost per task（每个任务成本）**：典型范围是每个子代理 $0.05-$0.15，取决于任务

## Troubleshooting（故障排除）

### Skills Not Loading（技能未加载）

**问题**：运行无头测试时找不到技能

**解决方案**：
1. 确保你从 superpowers 目录运行：`cd /path/to/superpowers && tests/...`
2. 检查 `~/.claude/settings.json` 的 `enabledPlugins` 中有 `"superpowers@superpowers-dev": true`
3. 验证技能存在于 `skills/` 目录中

### Permission Errors（权限错误）

**问题**：Claude 被阻止写入文件或访问目录

**解决方案**：
1. 使用 `--permission-mode bypassPermissions` 标志
2. 使用 `--add-dir /path/to/temp/dir` 授予对测试目录的访问权限
3. 检查测试目录的文件权限

### Test Timeouts（测试超时）

**问题**：测试耗时过长并超时

**解决方案**：
1. 增加超时时间：`timeout 1800 claude ...`（30 分钟）
2. 检查技能逻辑中是否有无限循环
3. 审查子代理任务复杂度

### Session File Not Found（会话文件未找到）

**问题**：测试运行后找不到会话记录

**解决方案**：
1. 检查 `~/.claude/projects/` 中正确的项目目录
2. 使用 `find ~/.claude/projects -name "*.jsonl" -mmin -60` 查找最近的会话
3. 验证测试确实运行了（检查测试输出中的错误）

## Writing New Integration Tests（编写新的集成测试）

### Template（模板）

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/test-helpers.sh"

# 创建测试项目
TEST_PROJECT=$(create_test_project)
trap "cleanup_test_project $TEST_PROJECT" EXIT

# 设置测试文件...
cd "$TEST_PROJECT"

# 使用技能运行 Claude
PROMPT="Your test prompt here"
cd "$SCRIPT_DIR/../.." && timeout 1800 claude -p "$PROMPT" \
  --allowed-tools=all \
  --add-dir "$TEST_PROJECT" \
  --permission-mode bypassPermissions \
  2>&1 | tee output.txt

# 查找并分析会话
WORKING_DIR_ESCAPED=$(echo "$SCRIPT_DIR/../.." | sed 's/\\//-/g' | sed 's/^-//')
SESSION_DIR="$HOME/.claude/projects/$WORKING_DIR_ESCAPED"
SESSION_FILE=$(find "$SESSION_DIR" -name "*.jsonl" -type f -mmin -60 | sort -r | head -1)

# 通过解析会话记录验证行为
if grep -q '"name":"Skill".*"skill":"your-skill-name"' "$SESSION_FILE"; then
    echo "[PASS] Skill was invoked"
fi

# 显示 token 分析
python3 "$SCRIPT_DIR/analyze-token-usage.py" "$SESSION_FILE"
```

### Best Practices（最佳实践）

1. **始终清理**：使用 trap 清理临时目录
2. **解析记录**：不要 grep 用户可见的输出——解析 `.jsonl` 会话文件
3. **授予权限**：使用 `--permission-mode bypassPermissions` 和 `--add-dir`
4. **从插件目录运行**：技能只在从 superpowers 目录运行时加载
5. **显示 token 使用量**：始终包含 token 分析以提高成本可见性
6. **测试真实行为**：验证实际创建的文件、通过的测试、做出的提交

## Session Transcript Format（会话记录格式）

会话记录是 JSONL（JSON Lines）文件，每行是一个表示消息或工具结果的 JSON 对象。

### Key Fields（关键字段）

```json
{
  "type": "assistant",
  "message": {
    "content": [...],
    "usage": {
      "input_tokens": 27,
      "output_tokens": 3996,
      "cache_read_input_tokens": 1213703
    }
  }
}
```

### Tool Results（工具结果）

```json
{
  "type": "user",
  "toolUseResult": {
    "agentId": "3380c209",
    "usage": {
      "input_tokens": 2,
      "output_tokens": 787,
      "cache_read_input_tokens": 24989
    },
    "prompt": "You are implementing Task 1...",
    "content": [{"type": "text", "text": "..."}]
  }
}
```

`agentId` 字段链接到子代理会话，`usage` 字段包含该特定子代理调用的 token 使用量。
