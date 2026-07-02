---
name: security-and-hardening
description: 硬化代码防御漏洞。在处理用户输入、认证、数据存储或外部集成时使用。在构建任何接受不可信数据、管理用户会话或与第三方服务交互的功能时使用。
---

# 安全与硬化

## 概述

面向 Web 应用的安全优先开发实践。将每个外部输入视为敌对，每个 secret 视为神圣，每个授权检查视为强制。安全不是一个阶段——它是对每一行触及用户数据、认证或外部系统的代码的约束。

## 何时使用

- 构建任何接受用户输入的内容
- 实现认证或授权
- 存储或传输敏感数据
- 与外部 API 或服务集成
- 添加文件上传、webhook 或回调
- 处理支付或 PII 数据

## 流程：先做威胁建模

没有威胁模型的控制措施只是为了猜测。在做硬化之前，花五分钟像攻击者一样思考：

1. **映射信任边界。** 不可信数据从哪里进入你的系统？HTTP 请求、表单字段、文件上传、webhook、第三方 API、消息队列和 **LLM 输出**。每条边界都是攻击面。
2. **命名资产。** 什么值得窃取或破坏？凭证、PII、支付数据、管理员操作、资金流转。
3. **在每条边界上运行 STRIDE**——一个快速视角，不是形式化流程：

| 威胁 | 问自己 | 典型缓解措施 |
|---|---|---|
| **S**poofing（假冒） | 有人可以冒充用户/服务吗？ | 认证、签名验证 |
| **T**ampering（篡改） | 数据在传输或存储时能否被篡改？ | 完整性检查、参数化查询、HTTPS |
| **R**epudiation（抵赖） | 操作能否后来被否认？ | 安全事件的审计日志 |
| **I**nformation disclosure（信息泄露） | 数据能否泄露？ | 加密、字段白名单、通用错误消息 |
| **D**enial of service（拒绝服务） | 能否被压垮？ | 速率限制、输入大小限制、超时 |
| **E**levation of privilege（权限提升） | 用户能否获得不应有的权限？ | 授权检查、最小权限 |

4. **在用例旁边写滥用案例。** 对于每个功能，问"我怎样滥用它？"——然后让那个成为你的第一个测试。

如果你不能为一个功能命名信任边界，你就没有准备好保护它。这是 OWASP **A04: Insecure Design**——大多数数据泄露始于设计，而非代码。

## 三层边界系统

### 始终做（无例外）

- **在系统边界验证所有外部输入**（API 路由、表单处理程序）
- **参数化所有数据库查询**——永远不要将用户输入拼接到 SQL 中
- **编码输出**以防止 XSS（使用框架的自动转义，不要绕过它）
- **对所有外部通信使用 HTTPS**
- **使用 bcrypt/scrypt/argon2 哈希密码**（永远不要存储明文）
- **设置安全头**（CSP、HSTS、X-Frame-Options、X-Content-Type-Options）
- **为会话使用 httpOnly、secure、sameSite Cookie**
- **每次发布前运行 `npm audit`**（或等效命令）

### 先询问（需要人工批准）

- 添加新的认证流程或更改认证逻辑
- 存储新类别的敏感数据（PII、支付信息）
- 添加新的外部服务集成
- 更改 CORS 配置
- 添加文件上传处理程序
- 修改速率限制或节流
- 授予提升的权限或角色

### 永远不做

- **永远不要将 secrets 提交到版本控制**（API 密钥、密码、tokens）
- **永远不要记录敏感数据**（密码、tokens、完整信用卡号）
- **永远不要信任客户端验证**作为安全边界
- **永远不要为了方便禁用安全头**
- **永远不要对用户提供的数据使用 `eval()` 或 `innerHTML`**
- **永远不要在客户端可访问的存储中存储会话**（localStorage 中的认证 tokens）
- **永远不要向用户暴露堆栈跟踪**或内部错误详情

## OWASP Top 10 预防模式

这些是预防模式，不是排名。关于 2021 排序，参见 `references/security-checklist.md` 中的快速参考表。

### 注入（SQL、NoSQL、OS 命令）

```typescript
// BAD: 通过字符串拼接的 SQL 注入
const query = `SELECT * FROM users WHERE id = '${userId}'`;

// GOOD: 参数化查询
const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);

// GOOD: 带参数化输入的 ORM
const user = await prisma.user.findUnique({ where: { id: userId } });
```

### 失效的认证

```typescript
// 密码哈希
import { hash, compare } from 'bcrypt';

const SALT_ROUNDS = 12;
const hashedPassword = await hash(plaintext, SALT_ROUNDS);
const isValid = await compare(plaintext, hashedPassword);

// 会话管理
app.use(session({
  secret: process.env.SESSION_SECRET,  // 来自环境变量，而非代码
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,     // 不可通过 JavaScript 访问
    secure: true,       // 仅 HTTPS
    sameSite: 'lax',    // CSRF 保护
    maxAge: 24 * 60 * 60 * 1000,  // 24 小时
  },
}));
```

### 跨站脚本 (XSS)

```typescript
// BAD: 将用户输入渲染为 HTML
element.innerHTML = userInput;

// GOOD: 使用框架的自动转义（React 默认如此）
return <div>{userInput}</div>;

// 如果你必须渲染 HTML，先做清理
import DOMPurify from 'dompurify';
const clean = DOMPurify.sanitize(userInput);
```

### 失效的访问控制

```typescript
// 始终检查授权，而不仅仅是认证
app.patch('/api/tasks/:id', authenticate, async (req, res) => {
  const task = await taskService.findById(req.params.id);

  // 检查已认证用户拥有此资源
  if (task.ownerId !== req.user.id) {
    return res.status(403).json({
      error: { code: 'FORBIDDEN', message: '无权修改此任务' }
    });
  }

  // 继续更新
  const updated = await taskService.update(req.params.id, req.body);
  return res.json(updated);
});
```

### 安全配置错误

```typescript
// 安全头（使用 helmet for Express）
import helmet from 'helmet';
app.use(helmet());

// 内容安全策略
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],
    styleSrc: ["'self'", "'unsafe-inline'"],  // 可能的话收紧
    imgSrc: ["'self'", 'data:', 'https:'],
    connectSrc: ["'self'"],
  },
}));

// CORS — 限制为已知来源
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true,
}));
```

### 敏感数据暴露

```typescript
// 永远不要在 API 响应中返回敏感字段
function sanitizeUser(user: UserRecord): PublicUser {
  const { passwordHash, resetToken, ...publicFields } = user;
  return publicFields;
}

// 使用环境变量存储 secrets
const API_KEY = process.env.STRIPE_API_KEY;
if (!API_KEY) throw new Error('STRIPE_API_KEY 未配置');
```

### 服务端请求伪造 (SSRF)

每当服务器获取一个用户影响的 URL 时——webhook、"从 URL 导入"、图片代理、链接预览——攻击者可以将其指向内部服务（云元数据、`localhost`、私有 IP）。

```typescript
// BAD: 获取用户给你的任何 URL
await fetch(req.body.webhookUrl);

// GOOD: 白名单 scheme + host，如果解析到的任何 IP 是私有的则拒绝，禁止重定向
import { lookup } from 'node:dns/promises';
import ipaddr from 'ipaddr.js';

const ALLOWED_HOSTS = new Set(['hooks.example.com']);

async function assertSafeUrl(raw: string): Promise<URL> {
  const url = new URL(raw);
  if (url.protocol !== 'https:') throw new Error('仅支持 https');
  if (!ALLOWED_HOSTS.has(url.hostname)) throw new Error('host 不在白名单中');
  // 解析所有记录；单个私有/保留地址即导致检查失败。
  const addrs = await lookup(url.hostname, { all: true });
  if (addrs.some((a) => ipaddr.parse(a.address).range() !== 'unicast')) {
    throw new Error('私有/保留 IP');
  }
  return url;
}

await fetch(await assertSafeUrl(req.body.webhookUrl), { redirect: 'error' });
```

`range() !== 'unicast'` 检查覆盖了回环、链路本地 `169.254.169.254`（云元数据，SSRF 的第一目标）、私有和唯一本地区间，跨 IPv4 和 IPv6。

**警告——这仍有一个 TOCTOU 缺口。** `fetch` 在检查后再次解析 DNS，因此使用短 TTL 记录的攻击者可以在验证和连接之间重绑定到内网 IP。对于高风险面，解析一次并连接到固定的 IP，或在前端放置一个过滤代理（`request-filtering-agent` / `ssrf-req-filter`）。

## 输入验证模式

### 边界的模式验证

```typescript
import { z } from 'zod';

const CreateTaskSchema = z.object({
  title: z.string().min(1).max(200).trim(),
  description: z.string().max(2000).optional(),
  priority: z.enum(['low', 'medium', 'high']).default('medium'),
  dueDate: z.string().datetime().optional(),
});

// 在路由处理程序中验证
app.post('/api/tasks', async (req, res) => {
  const result = CreateTaskSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: '无效输入',
        details: result.error.flatten(),
      },
    });
  }
  // result.data 现在已被类型化和验证
  const task = await taskService.create(result.data);
  return res.status(201).json(task);
});
```

### 文件上传安全

```typescript
// 限制文件类型和大小
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp'];
const MAX_SIZE = 5 * 1024 * 1024; // 5MB

function validateUpload(file: UploadedFile) {
  if (!ALLOWED_TYPES.includes(file.mimetype)) {
    throw new ValidationError('不允许的文件类型');
  }
  if (file.size > MAX_SIZE) {
    throw new ValidationError('文件过大（最大 5MB）');
  }
  // 不要信任文件扩展名——如果关键，检查 magic bytes
}
```

## 分类 npm audit 结果

并非所有审计发现都需要立即操作。使用这个决策树：

```
npm audit 报告了一个漏洞
├── Severity: critical（严重）或 high（高）
│   ├── 漏洞代码在你的应用中是否可到达？
│   │   ├── 是 --> 立即修复（更新、打补丁或替换依赖）
│   │   └── 否（仅开发依赖、未使用的代码路径） --> 尽快修复，但不阻塞
│   └── 是否有可用的修复？
│       ├── 是 --> 更新到已修复版本
│       └── 否 --> 检查变通方案，考虑替换依赖，或加入白名单并设置审查日期
├── Severity: moderate（中等）
│   ├── 在生产中可到达？ --> 在下一个发布周期中修复
│   └── 仅开发依赖？ --> 方便时修复，在 backlog 中跟踪
└── Severity: low（低）
    └── 在常规依赖更新中跟踪和修复
```

**关键问题：**
- 漏洞函数是否在你的代码路径中被实际调用？
- 依赖是运行时依赖还是仅开发依赖？
- 考虑到你的部署上下文，该漏洞是否可被利用（例如，一个客户端应用中的服务端漏洞）？

当你推迟修复时，记录原因并设置审查日期。

### 供应链卫生

`npm audit` 捕获已知的 CVE；它不会捕获恶意或拼写劫持的包。此外：

- **提交 lockfile** 并在 CI 中使用 `npm ci`（而非 `npm install`）安装——可复现的构建，不会静默版本漂移。
- **在添加之前审查新的依赖**——维护情况、下载量以及它们是否真正值得引入。每个依赖都是攻击面（OWASP **A06: Vulnerable Components**、**LLM03: Supply Chain**）。
- **警惕不熟悉包中的 `postinstall` 脚本**——它们会在安装时运行任意代码。
- **关注拼写劫持**——`cross-env` vs `crossenv`、`react-dom` vs `reactdom`。

## 速率限制

```typescript
import rateLimit from 'express-rate-limit';

// 通用 API 速率限制
app.use('/api/', rateLimit({
  windowMs: 15 * 60 * 1000, // 15 分钟
  max: 100,                   // 每个窗口 100 次请求
  standardHeaders: true,
  legacyHeaders: false,
}));

// 认证端点更严格的限制
app.use('/api/auth/', rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,  // 每 15 分钟 10 次尝试
}));
```

## Secrets 管理

```
.env 文件:
  ├── .env.example  → 已提交（带占位符值的模板）
  ├── .env          → 不提交（包含真实 secrets）
  └── .env.local    → 不提交（本地覆盖）

.gitignore 必须包含:
  .env
  .env.local
  .env.*.local
  *.pem
  *.key
```

**提交前始终检查：**
```bash
# 检查意外暂存的 secrets
git diff --cached | grep -i "password\|secret\|api_key\|token"
```

**如果某个 secret 被提交过，轮换它。** 删除该行或重写历史是不够的——假设它到达远程仓库那一刻就已泄露。先撤销并重新发放密钥，再从历史中清除。

## 保护 AI / LLM 功能

如果你的应用调用 LLM——聊天机器人、摘要器、agent、RAG——它继承了一个新的攻击面。将其映射到 [OWASP Top 10 for LLM Applications (2025)](https://genai.owasp.org/llm-top-10/)：

- **将所有模型输出视为不可信输入（LLM05: Improper Output Handling）。** 永远不要将 LLM 输出直接传入 `eval`、SQL、shell、`innerHTML` 或文件路径。像验证原始用户输入一样验证和编码它。
- **假设提示可被劫持（LLM01: Prompt Injection）。** 上下文窗口中的不可信文本——用户消息、抓取的网页、PDF——可以携带指令。系统提示不是安全边界；在代码中强制执行权限，而非在提示中。
- **不要将 secrets 和其他用户数据放入提示（LLM02 / LLM07）。** 上下文中的任何东西都可能被回显。不要把 API 密钥、跨租户数据或完整的系统提示放在模型可以重复它的地方。
- **约束工具和 agent 权限（LLM06: Excessive Agency）。** 将工具范围缩小到最小，对破坏性或不可逆操作要求确认，并验证每个工具参数。
- **限制消费（LLM10: Unbounded Consumption）。** 限制 tokens、请求速率和循环/递归深度，使得精心构造的输入不能增加成本或使系统挂起。
- **隔离检索数据（LLM08: Vector and Embedding Weaknesses）。** 在 RAG 中，将向量存储视为信任边界：按租户分区嵌入，使一个用户无法检索到另一个用户的数据，并在索引前验证文档，使得被污染的内容不能引导答案。

```typescript
// BAD: 信任模型输出作为命令或标记
const sql = await llm.generate(`为以下内容编写 SQL: ${userQuestion}`);
await db.query(sql);                                   // 任意查询执行
container.innerHTML = await llm.reply(userMessage);   // 存储型 XSS，通过模型

// GOOD: 模型输出是数据——防御性解析，然后验证，然后编码
let intent;
try {
  intent = CommandSchema.parse(JSON.parse(await llm.replyJson(userMessage)));
} catch {
  throw new ValidationError('意外的模型输出'); // JSON.parse 或 schema 失败
}
await runAllowlistedAction(intent.action, intent.params);
container.textContent = await llm.reply(userMessage);
```

## 安全审查检查清单

```markdown
### Authentication（认证）
- [ ] 密码使用 bcrypt/scrypt/argon2 哈希（salt rounds ≥ 12）
- [ ] 会话 tokens 是 httpOnly、secure、sameSite
- [ ] 登录有速率限制
- [ ] 密码重置 tokens 会过期

### Authorization（授权）
- [ ] 每个端点检查用户权限
- [ ] 用户只能访问自己的资源
- [ ] 管理员操作需要管理员角色验证

### Input（输入）
- [ ] 所有用户输入在边界验证
- [ ] SQL 查询是参数化的
- [ ] HTML 输出是编码/转义的
- [ ] 服务端 URL 获取使用白名单（无 SSRF 到内部服务）

### Data（数据）
- [ ] 代码或版本控制中没有 secrets
- [ ] 敏感字段从 API 响应中排除
- [ ] PII 在静态存储中加密（如果适用）

### Infrastructure（基础设施）
- [ ] 安全头已配置（CSP、HSTS 等）
- [ ] CORS 限制为已知来源
- [ ] 依赖已审计漏洞
- [ ] 错误消息不暴露内部信息

### Supply Chain（供应链）
- [ ] Lockfile 已提交；CI 使用 `npm ci` 安装
- [ ] 新依赖已审查（维护情况、下载量、postinstall 脚本）

### AI / LLM（如果使用）
- [ ] 模型输出被视为不可信（无 eval/SQL/innerHTML/shell）
- [ ] Secrets 和其他用户数据未放入提示
- [ ] 工具/agent 权限已限定范围；破坏性操作要求确认
```
## 另请参阅

关于详细的安全检查清单和提交前验证步骤，参见 `references/security-checklist.md`。

## 常见借口

| 借口 | 现实 |
|---|---|
| "这是内部工具，安全不重要" | 内部工具会被入侵。攻击者瞄准最薄弱的一环。 |
| "我们后面再加安全" | 安全改造比内建安全难 10 倍。现在就加上。 |
| "没人会利用这个" | 自动扫描器会发现它。安全靠模糊性不是安全。 |
| "框架会处理安全问题" | 框架提供工具，而非保证。你仍然需要正确使用它们。 |
| "这只是原型" | 原型变成生产。从第一天起就要有安全习惯。 |
| "威胁建模在这里有点过度" | 五分钟的"我怎样攻击它？"能防止以后任何控制措施都无法修补的设计缺陷。 |
| "这只是一个 LLM 输出，只是文本" | 那个"文本"可能是 SQL 语句、script 标签或 shell 命令。像对待任何不可信输入一样对待它。 |

## 警示信号

- 用户输入直接传入数据库查询、shell 命令或 HTML 渲染
- Secrets 在源代码或提交历史中
- API 端点没有认证或授权检查
- 缺少 CORS 配置或通配符（`*`）来源
- 认证端点没有速率限制
- 堆栈跟踪或内部错误暴露给用户
- 依赖有已知的严重漏洞
- 服务器获取用户提供的 URL 但没有白名单（SSRF）
- LLM/模型输出传入查询、DOM、shell 或 `eval`
- Secrets、PII 或完整的系统提示放置在 LLM 上下文窗口中

## 验证

实现安全相关代码后：

- [ ] `npm audit` 显示没有严重或高危漏洞
- [ ] 源代码或 git 历史中没有 secrets
- [ ] 所有用户输入在系统边界验证
- [ ] 每个受保护端点上检查了认证和授权
- [ ] 响应中存在安全头（用浏览器 DevTools 检查）
- [ ] 错误响应不暴露内部详情
- [ ] 认证端点活跃速率限制
- [ ] 服务端 URL 获取对照白名单验证（无 SSRF）
- [ ] LLM/模型输出在使用前验证和编码（如果存在 AI 功能）
