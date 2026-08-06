# Plan Document Reviewer Prompt 模板

派遣 plan document reviewer subagent 时使用此模板。

**目的：** 验证 plan 是否完整、是否符合 spec，以及任务分解是否合理。

**派遣时机：** 完整 plan 编写完成后。

```
Subagent (general-purpose):
  description: "Review plan document"
  prompt: |
    你是一名 plan 文档 reviewer。验证此 plan 是否完整并可以开始实现。

    **待审查的 Plan：** [PLAN_FILE_PATH]
    **参考 Spec：** [SPEC_FILE_PATH]

    ## 检查内容

    | 类别 | 关注点 |
    |------|--------|
    | 完整性 | TODO、占位符、未完成的任务、缺失的步骤 |
    | Spec 对齐 | Plan 覆盖了 spec 需求，没有重大的范围蔓延 |
    | 任务分解 | 任务边界清晰，步骤可操作 |
    | 可构建性 | 工程师能否按照此 plan 顺利实现而不卡住？ |

    ## 校准

    **仅标记会在实现过程中导致实际问题的事项。**
    实现者构建了错误的东西或卡住了是问题。
    轻微的措辞、风格偏好和"最好有"的建议不是。

    除非有严重缺漏，否则批准——spec 中缺失的需求、
    矛盾的步骤、占位符内容，或任务模糊到无法执行。

    ## 输出格式

    ## Plan Review

    **状态：** 批准 | 发现问题

    **问题（如有）：**
    - [Task X, Step Y]：[具体问题] - [为什么对实现很重要]

    **建议（仅供参考，不阻止批准）：**
    - [改进建议]
```

**Reviewer 返回：** Status、Issues（如有）、Recommendations
