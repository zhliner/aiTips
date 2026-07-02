# Observability Checklist（可观测性检查清单）

生产代码埋点快速参考。配合 `observability-and-instrumentation` skill 使用。

## 目录

- [On-Call Questions（On-Call 问题——从这里开始）](#on-call-question-on-call-问题从这里开始)
- [Structured Logging（结构化日志）](#structured-logging结构化日志)
- [Metrics（指标）](#metrics指标)
- [Distributed Tracing（分布式追踪）](#distributed-tracing分布式追踪)
- [Alerting（告警）](#alerting告警)
- [Dashboards（仪表盘）](#dashboards仪表盘)
- [Verify the Telemetry（验证遥测）](#verify-the-telemetry验证遥测)
- [Pre-Launch Gate（发布前关卡）](#pre-launch-gate发布前关卡)

## On-Call Questions（On-Call 问题——从这里开始）

没有问题的遥测只是噪音。在做任何埋点之前：

- [ ] 写下 On-Call 工程师将会问的关于此功能的 2–4 个问题
- [ ] 下方的每个信号都映射到其中一个问题
- [ ] 每个问题与正确的信号类型匹配：指标说明*某处*出了问题，追踪说明*哪里*出了问题，日志说明*为什么*出了问题

## Structured Logging（结构化日志）

- [ ] 日志是结构化的（JSON），具有稳定的 event name——而非自由格式字符串
- [ ] 每条日志行都携带 correlation/request ID，在系统边界处生成或接受
- [ ] Correlation ID 在每次外呼调用和异步边界上传播（HTTP headers、队列元数据）
- [ ] 日志级别保持一致：`error` = 不变量被破坏，可能需要人工介入；`warn` = 降级但已处理；`info` = 重要的业务事件；`debug` = 生产环境关闭
- [ ] 日志行中无密钥、Token、密码或未脱敏的 PII（来自 `security-and-hardening` 的硬性规则）
- [ ] 字段使用 allowlist——无完整请求/响应体，无 auth headers
- [ ] 外部服务调用仅记录元数据：端点、状态、延迟、重试次数、脱敏的标识符
- [ ] 实际日志输出已抽查：结构化字段，而非 `[object Object]`

## Metrics（指标）

- [ ] 每个端点和每个外部依赖均已实施 **RED** 埋点：Rate（速率）、Errors（错误）、Duration（延迟）
- [ ] 每个资源（队列、连接池、主机）均已实施 **USE** 埋点：Utilization（利用率）、Saturation（饱和度）、Errors（错误）
- [ ] 延迟使用直方图；p50/p95/p99 可查询——绝不使用平均值
- [ ] 所有 label 来自小型固定集合（路由模板、状态码类别、provider 名称）
- [ ] 无无限制的 label 值：无 user ID、tenant ID、email、原始 URL、request ID 或错误消息文本
- [ ] 状态码按类别分组（`5xx`，而非 `503`）
- [ ] 每个 worker/queue 跟踪队列深度和处理延迟

## Distributed Tracing（分布式追踪）

- [ ] OpenTelemetry（或等价方案）在服务启动时、其他导入之前初始化
- [ ] HTTP、gRPC 和 DB 客户端已启用自动埋点
- [ ] Trace 上下文在每次外呼调用上传播（W3C `traceparent`/`tracestate`），并从每次入站请求中提取
- [ ] 上下文在异步边界上存活——队列消息携带 Trace 元数据
- [ ] 手动 Span 仅在有意义的内部工作单元上创建，并附带 On-Call 会用来过滤的属性
- [ ] Span 属性中无密钥或 PII
- [ ] Head-based 采样以低默认比例进行；若 Tail Sampling 可用，则保留 100% 的错误

## Alerting（告警）

- [ ] 每条告警基于症状（错误率、p99 延迟、队列积压时间）——原因（CPU、磁盘、重启）放到仪表盘，而非告警通知
- [ ] 每条告警可操作；对"忽略就好，会自动恢复"的告警予以删除
- [ ] 每条告警链接到 runbook——至少三行：含义、首要查询命令、升级路径
- [ ] 阈值和持续时长由 SLO 或历史数据论证，而非猜测
- [ ] 仅两个严重级别：**page**（影响用户，立即处理）和 **ticket**（降级，本周处理）
- [ ] 每条新告警测试触发一次：到达了正确的渠道，runbook 链接能正常工作
- [ ] 没有每天触发却被无操作确认的告警

## Dashboards（仪表盘）

- [ ] 服务健康仪表盘存在：错误率、p99 延迟、流量、饱和度
- [ ] 依赖健康面板按服务显示错误率和延迟
- [ ] 仪表盘能回答本清单顶部的 On-Call 问题——而非"除了答案什么都展示了"
- [ ] 默认时间范围合理（1h–6h，而非 30d）

## Verify the Telemetry（验证遥测）

埋点也是代码，可能出错：

- [ ] 在 staging 环境强制触发了一个错误 → 能通过 correlation ID 在日志中找到它
- [ ] 发送了测试流量 → 指标序列以预期的 label 和合理的值出现
- [ ] 在追踪 UI 中端到端跟踪了一个请求 → 没有断开的 Span
- [ ] 仅凭遥测就诊断出一次人为触发的故障，无需阅读源代码

## Pre-Launch Gate（发布前关卡）

在功能发布到生产环境之前，以下所有条件必须为真：

- [ ] 结构化日志流入日志聚合器
- [ ] 每个新端点和依赖的 RED 指标在仪表盘中可见
- [ ] 至少配置了一条基于症状的告警，附带 runbook，已测试触发
- [ ] 一个请求可以跨越它触及的每个服务被追踪
- [ ] On-Call 知道 runbook 在哪里

关于发布当天的监控顺序和回滚触发条件，参见 `shipping-and-launch` skill。
