# 基于条件的等待

## 概述

不稳定的测试往往使用任意延迟来猜测时序，这会产生竞态条件：测试在快速机器上通过，但在高负载或 CI 环境下则失败。

**核心原则：** 等待你关心的实际条件，而不是猜测它需要多长时间。

## 适用场景

```mermaid
flowchart TD
    A{测试使用了 setTimeout/sleep？}
    B{测试的是时序行为？}
    C[记录为什么需要超时]
    D[使用基于条件的等待]

    A -->|是| B
    B -->|是| C
    B -->|否| D
```

**适用情况：**
- 测试中存在任意延迟（`setTimeout`、`sleep`、`time.sleep()`）
- 测试不稳定（有时通过，负载下失败）
- 测试在并行运行时超时
- 等待异步操作完成

**不适用情况：**
- 测试实际的时序行为（防抖、节流间隔）
- 如果使用了任意超时，始终记录原因

## 核心模式

```typescript
// ❌ 之前：猜测时序
await new Promise(r => setTimeout(r, 50));
const result = getResult();
expect(result).toBeDefined();

// ✅ 之后：等待条件
await waitFor(() => getResult() !== undefined);
const result = getResult();
expect(result).toBeDefined();
```

## 快速模式

| 场景 | 模式 |
|----------|---------|
| 等待事件 | `waitFor(() => events.find(e => e.type === 'DONE'))` |
| 等待状态 | `waitFor(() => machine.state === 'ready')` |
| 等待数量 | `waitFor(() => items.length >= 5)` |
| 等待文件 | `waitFor(() => fs.existsSync(path))` |
| 复杂条件 | `waitFor(() => obj.ready && obj.value > 10)` |

## 实现

通用轮询函数：
```typescript
async function waitFor<T>(
  condition: () => T | undefined | null | false,
  description: string,
  timeoutMs = 5000
): Promise<T> {
  const startTime = Date.now();

  while (true) {
    const result = condition();
    if (result) return result;

    if (Date.now() - startTime > timeoutMs) {
      throw new Error(`Timeout waiting for ${description} after ${timeoutMs}ms`);
    }

    await new Promise(r => setTimeout(r, 10)); // 每 10ms 轮询一次
  }
}
```

本目录下的 `condition-based-waiting-example.ts` 包含来自实际调试会话的完整实现，其中包含领域特定的辅助函数（`waitForEvent`、`waitForEventCount`、`waitForEventMatch`）。

## 常见错误

**❌ 轮询太快：** `setTimeout(check, 1)` — 浪费 CPU
**✅ 修复：** 每 10ms 轮询一次

**❌ 没有超时：** 条件永远不满足则无限循环
**✅ 修复：** 始终包含超时，并提供清晰的错误信息

**❌ 数据过时：** 在循环之前缓存状态
**✅ 修复：** 在循环内部调用 getter 获取最新数据

## 任意超时何时是正确的

```typescript
// 工具每 100ms tick 一次 — 需要 2 个 tick 来验证部分输出
await waitForEvent(manager, 'TOOL_STARTED'); // 首先：等待条件
await new Promise(r => setTimeout(r, 200));   // 然后：等待时序行为
// 200ms = 以 100ms 为间隔的 2 个 tick — 已记录且合理
```

**要求：**
1. 首先等待触发条件
2. 基于已知的时序（而非猜测）
3. 注释解释原因

## 实际效果

来自调试会话（2025-10-03）：
- 修复了 3 个文件中的 15 个不稳定测试
- 通过率：60% → 100%
- 执行时间：加快 40%
- 不再有竞态条件
