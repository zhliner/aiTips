---
name: internal-comms
description: 一组帮助撰写各类内部沟通文档的资源，采用公司常用的格式。当被要求撰写某种内部沟通文档时（状态报告、领导层更新、3P 更新、公司通讯、FAQ、事故报告、项目更新等），Claude 应使用此技能。
license: Complete terms in LICENSE.txt
---

## 何时使用此技能
撰写内部沟通文档时，可在以下场景使用此技能：
- 3P 更新（Progress、Plans、Problems）
- 公司通讯
- FAQ 回复
- 状态报告
- 领导层更新
- 项目更新
- 事故报告

## 如何使用此技能

撰写任何内部沟通文档时：

1. **从请求中识别沟通文档类型**
2. **从 `examples/` 目录加载相应的指南文件**：
    - `examples/3p-updates.md` - 用于 Progress/Plans/Problems 团队更新
    - `examples/company-newsletter.md` - 用于全公司范围的通讯
    - `examples/faq-answers.md` - 用于回答常见问题
    - `examples/general-comms.md` - 用于不属于上述类型的其他内部沟通
3. **按照该文件中的具体说明**处理格式、语气和内容收集

如果沟通文档类型与现有指南均不匹配，请询问更多细节或了解期望的格式。

## 关键词
3P updates, company newsletter, company comms, weekly update, faqs, common questions, updates, internal comms
