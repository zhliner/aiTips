# 可观测性检查清单

在生产环境代码中进行埋点的快速参考。可与 `observability-and-instrumentation` skill 配合使用。

## 目录

- [On-Call 问题（从这里开始）](#on-call-问题从这里开始)
- [结构化日志](#结构化日志)
- [指标（Metrics）](#指标metrics)
- [分布式追踪](#分布式追踪)
- [告警](#告警)
- [仪表盘](#仪表盘)
- [验证遥测数据](#验证遥测数据)
- [上线前检查点](#上线前检查点)

## On-Call 问题（从这里开始）

没有明确问题的遥测数据就是噪音。在进行任何埋点之前：

- [ ] 已写下 2–4 个 on-call 工程师会针对该功能提出的问题
- [ ] 下方每一项信号都能对应到其中一个问题
- [ ] 每个问题都匹配了正确的信号类型：metrics 说明**有**问题，traces 说明问题**在哪**，logs 说明**为什么**出问题

## 结构化日志

- [ ] 日志采用结构化格式（JSON），具有稳定的事件名称——而非自由文本字符串
- [ ] 每条日志都携带 correlation/request ID，该 ID 在系统边界处生成或接收
- [ ] Correlation ID 在每次出站调用和异步边界处进行传播（HTTP headers、queue metadata）
- [ ] 日志级别保持一致：`error` = 不变量被破坏，可能需要人工介入；`warn` = 功能降级但已处理；`info` = 重要业务事件；`debug` = 生产环境中关闭
- [ ] 任何日志行中都不包含 secrets、tokens、passwords 或未脱敏的 PII（来自 `security-and-hardening` 的硬性规则）
- [ ] 字段采用白名单机制——不记录完整的 request/response bodies，不记录 auth headers
- [ ] 外部服务调用仅记录元数据：endpoint、status、latency、attempt count、已脱敏的 identifiers
- [ ] 实际日志输出经过抽查：是结构化字段，而非 `[object Object]`

## 指标（Metrics）

- [ ] 每个 endpoint 和每个外部依赖都已接入 **RED** 指标：Rate、Errors、Duration
- [ ] 每个资源（queues、pools、hosts）都已接入 **USE** 指标：Utilization、Saturation、Errors
- [ ] Latency 使用 histogram 表示；p50/p95/p99 可查询——绝不使用平均值
- [ ] 所有 label 均来自有限且固定的集合（route template、status class、provider name）
- [ ] 不存在无界的 label 值：不包含 user IDs、tenant IDs、emails、raw URLs、request IDs 或 error message 文本
- [ ] Status codes 按类别分组（使用 `5xx`，而非 `503`）
- [ ] 每个 worker/queue 都追踪了 queue depth 和 processing duration

## 分布式追踪

- [ ] OpenTelemetry（或同等方案）在服务启动时初始化，且先于其他 imports
- [ ] 已为 HTTP、gRPC 和 DB clients 启用自动埋点
- [ ] Trace context 在每次出站调用时传播（W3C `traceparent`/`tracestate`），并从每个入站请求中提取
- [ ] Context 能够跨越异步边界——queue messages 携带 trace metadata
- [ ] 手动 span 仅围绕有意义的内部工作单元创建，并附带 on-call 人员用于筛选的 attributes
- [ ] span attributes 中不包含 secrets 或 PII
- [ ] 采用基于 header 的采样，默认使用较低采样率；若支持 tail sampling，则 100% 保留 errors

## 告警

- [ ] 每条告警都基于症状（error rate、p99 latency、queue age）——原因类指标（CPU、disk、restarts）放在 dashboards 中，而非触发 pager
- [ ] 每条告警都是可执行的；"忽略它，会自动恢复"类的告警已被删除
- [ ] 每条告警都链接到 runbook——至少包含三行内容：含义说明、首先要执行的查询、升级路径
- [ ] 阈值和持续时间由 SLO 或历史数据作为依据，而非凭猜测
- [ ] 仅有两个严重级别：**page**（影响用户，需立即处理）和 **ticket**（功能降级，本周内处理）
- [ ] 每条新告警都经过一次测试触发：确认到达了正确的 channel，且 runbook 链接可用
- [ ] 不存在每天触发但只是被确认而未采取行动的告警

## 仪表盘

- [ ] 服务健康仪表盘已存在：error rate、latency p99、traffic、saturation
- [ ] 依赖健康面板展示各服务的 error rates 和 latency
- [ ] 仪表盘能回答本清单顶部的 on-call 问题——而非"除了答案以外的所有内容"
- [ ] 默认时间范围合理（1h–6h，而非 30d）

## 验证遥测数据

埋点本身也是代码，可能会出错：

- [ ] 在 staging 中强制触发一个错误 → 通过 correlation ID 在日志中找到了它
- [ ] 发送测试流量 → metric series 以预期的 label 和合理的值出现
- [ ] 在 tracing UI 中端到端追踪了一个请求 → 没有断裂的 span
- [ ] 仅通过遥测数据就诊断出了一个人为制造的故障，无需阅读源码

## 上线前检查点

在功能发布到生产环境之前，以下所有条件必须满足：

- [ ] 结构化日志已流入 log aggregator
- [ ] 每个新 endpoint 和依赖的 RED metrics 在 dashboards 中可见
- [ ] 至少配置了一条基于症状的告警，附带 runbook，并经过测试触发
- [ ] 一个请求能够被跨所有涉及的服务进行完整追踪
- [ ] On-call 人员知道 runbook 的位置

关于上线当天的监控流程和回滚触发条件，请参阅 `shipping-and-launch` skill。
