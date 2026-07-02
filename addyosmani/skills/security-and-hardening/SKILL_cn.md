---
name: security-and-hardening
description: 加固代码以抵御漏洞。在处理用户输入、身份认证、数据存储或外部集成时使用。在构建任何接受不可信数据、管理用户会话或与第三方服务交互的功能时使用。
---

# 安全与加固（Security and Hardening）

## 概述

面向 Web 应用的安全优先开发实践。将每一个外部输入视为恶意的，每一个密钥视为神圣不可侵犯的，每一项授权检查视为强制性的。安全不是一个阶段——它是每一行涉及用户数据、身份认证或外部系统的代码都必须遵守的约束。

## 何时使用

- 构建任何接受用户输入的功能
- 实现身份认证（authentication）或授权（authorization）
- 存储或传输敏感数据
- 与外部 API 或服务集成
- 添加文件上传、webhook 或回调
- 处理支付或 PII 数据

## 流程：先进行威胁建模

没有威胁模型就附加的安全控制只是猜测。在加固之前，花五分钟时间以攻击者的思维来思考：

1. **绘制信任边界。** 不可信数据在哪里跨越边界进入你的系统？HTTP 请求、表单字段、文件上传、webhook、第三方 API、消息队列，以及 **LLM 输出**。每一个边界都是攻击面。
2. **明确资产。** 什么值得被窃取或破坏？凭据、PII、支付数据、管理员操作、资金流转。
3. **对每个边界运行 STRIDE 分析** ——这是一个快速视角，而非繁文缛节：

| 威胁 | 问题 | 典型缓解措施 |
|---|---|---|
| **S**poofing（伪造） | 有人能冒充用户/服务吗？ | 身份认证、签名验证 |
| **T**ampering（篡改） | 数据在传输中或存储时能被篡改吗？ | 完整性检查、参数化查询、HTTPS |
| **R**epudiation（抵赖） | 某个操作能事后被否认吗？ | 安全事件的审计日志 |
| **I**nformation disclosure（信息泄露） | 数据会泄露吗？ | 加密、字段白名单、通用错误信息 |
| **D**enial of service（拒绝服务） | 系统会被压垮吗？ | 速率限制、输入大小上限、超时设置 |
| **E**levation of privilege（权限提升） | 用户能获取不该有的权限吗？ | 授权检查、最小权限原则 |

4. **在用例旁边编写滥用用例。** 对于每个功能，问自己"我会如何滥用这个功能？"——然后将其作为你的第一个测试。

如果你无法说出某个功能的信任边界，说明你还没有准备好对其进行安全加固。这就是 OWASP **A04: Insecure Design** ——大多数安全漏洞始于设计阶段，而非代码阶段。

## 三级边界体系

### 必须执行（无例外）

- **在系统边界验证所有外部输入**（API 路由、表单处理器）
- **参数化所有数据库查询** ——永远不要将用户输入拼接到 SQL 中
- **编码输出** 以防止 XSS（使用框架的自动转义，不要绕过它）
- **使用 HTTPS** 进行所有外部通信
- **使用 bcrypt/scrypt/argon2 对密码进行哈希处理**（永远不要存储明文）
- **设置安全响应头**（CSP、HSTS、X-Frame-Options、X-Content-Type-Options）
- **使用 httpOnly、secure、sameSite cookies** 管理会话
- **在每次发布前运行 `npm audit`**（或等效工具）

### 需先确认（需要人工审批）

- 添加新的身份认证流程或修改认证逻辑
- 存储新类别的敏感数据（PII、支付信息）
- 添加新的外部服务集成
- 修改 CORS 配置
- 添加文件上传处理器
- 修改速率限制或限流策略
- 授予提升的权限或角色

### 禁止操作

- **永远不要将密钥提交到版本控制**（API 密钥、密码、token）
- **永远不要记录敏感数据**（密码、token、完整信用卡号）
- **永远不要将客户端验证作为安全边界**
- **永远不要为了方便而禁用安全响应头**
- **永远不要对用户提供的数据使用 `eval()` 或 `innerHTML`**
- **永远不要将会话存储在客户端可访问的存储中**（如用 localStorage 存储认证 token）
- **永远不要向用户暴露堆栈跟踪** 或内部错误详情

## OWASP Top 10 防护模式

这些是防护模式，而非排名。2021 年的排序请参见 `references/security-checklist.md` 中的快速参考表。

### 注入（SQL、NoSQL、OS 命令）

```typescript
// 错误：通过字符串拼接导致 SQL 注入
const query = `SELECT * FROM users WHERE id = '${userId}'`;

// 正确：参数化查询
const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);

// 正确：使用参数化输入的 ORM
const user = await prisma.user.findUnique({ where: { id: userId } });
```

### 身份认证失效（Broken Authentication）

```typescript
// 密码哈希处理
import { hash, compare } from 'bcrypt';

const SALT_ROUNDS = 12;
const hashedPassword = await hash(plaintext, SALT_ROUNDS);
const isValid = await compare(plaintext, hashedPassword);

// 会话管理
app.use(session({
  secret: process.env.SESSION_SECRET,  // 从环境变量获取，而非硬编码
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,     // 不可通过 JavaScript 访问
    secure: true,       // 仅 HTTPS
    sameSite: 'lax',    // CSRF 防护
    maxAge: 24 * 60 * 60 * 1000,  // 24 小时
  },
}));
```

### 跨站脚本攻击（XSS）

```typescript
// 错误：将用户输入作为 HTML 渲染
element.innerHTML = userInput;

// 正确：使用框架的自动转义（React 默认执行此操作）
return <div>{userInput}</div>;

// 如果必须渲染 HTML，请先进行净化处理
import DOMPurify from 'dompurify';
const clean = DOMPurify.sanitize(userInput);
```

### 访问控制失效（Broken Access Control）

```typescript
// 始终检查授权，而不仅仅是身份认证
app.patch('/api/tasks/:id', authenticate, async (req, res) => {
  const task = await taskService.findById(req.params.id);

  // 检查已认证用户是否拥有此资源
  if (task.ownerId !== req.user.id) {
    return res.status(403).json({
      error: { code: 'FORBIDDEN', message: 'Not authorized to modify this task' }
    });
  }

  // 执行更新操作
  const updated = await taskService.update(req.params.id, req.body);
  return res.json(updated);
});
```

### 安全配置错误（Security Misconfiguration）

```typescript
// 安全响应头（Express 使用 helmet）
import helmet from 'helmet';
app.use(helmet());

// Content Security Policy
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],
    styleSrc: ["'self'", "'unsafe-inline'"],  // 如果可能，进一步收紧
    imgSrc: ["'self'", 'data:', 'https:'],
    connectSrc: ["'self'"],
  },
}));

// CORS ——限制为已知来源
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true,
}));
```

### 敏感数据暴露（Sensitive Data Exposure）

```typescript
// 永远不要在 API 响应中返回敏感字段
function sanitizeUser(user: UserRecord): PublicUser {
  const { passwordHash, resetToken, ...publicFields } = user;
  return publicFields;
}

// 使用环境变量存储密钥
const API_KEY = process.env.STRIPE_API_KEY;
if (!API_KEY) throw new Error('STRIPE_API_KEY not configured');
```

### 服务端请求伪造（SSRF）

每当服务器获取用户影响的 URL 时——webhook、"从 URL 导入"、图片代理、链接预览——攻击者都可以将其指向内部服务（云元数据、`localhost`、私有 IP）。

```typescript
// 错误：直接请求用户提供的任意 URL
await fetch(req.body.webhookUrl);

// 正确：白名单 scheme + host，如果任何解析后的 IP 为私有地址则拒绝，禁止重定向
import { lookup } from 'node:dns/promises';
import ipaddr from 'ipaddr.js';

const ALLOWED_HOSTS = new Set(['hooks.example.com']);

async function assertSafeUrl(raw: string): Promise<URL> {
  const url = new URL(raw);
  if (url.protocol !== 'https:') throw new Error('https only');
  if (!ALLOWED_HOSTS.has(url.hostname)) throw new Error('host not allowed');
  // 解析所有记录；只要有一个私有/保留地址就未通过检查。
  const addrs = await lookup(url.hostname, { all: true });
  if (addrs.some((a) => ipaddr.parse(a.address).range() !== 'unicast')) {
    throw new Error('private/reserved IP');
  }
  return url;
}

await fetch(await assertSafeUrl(req.body.webhookUrl), { redirect: 'error' });
```

`range() !== 'unicast'` 检查覆盖了回环地址、链路本地地址 `169.254.169.254`（云元数据，SSRF 的首要目标）、私有地址和 IPv4/IPv6 的唯一本地地址范围。

**注意——这仍然存在 TOCTOU 缺陷。** `fetch` 在检查之后会再次解析 DNS，因此使用短 TTL 记录的攻击者可以在验证和连接之间将域名重新绑定到内部 IP。对于高风险场景，解析一次后直接连接到固定的 IP，或在前面放置过滤代理（`request-filtering-agent` / `ssrf-req-filter`）。

## 输入验证模式

### 边界处的 Schema 验证

```typescript
import { z } from 'zod';

const CreateTaskSchema = z.object({
  title: z.string().min(1).max(200).trim(),
  description: z.string().max(2000).optional(),
  priority: z.enum(['low', 'medium', 'high']).default('medium'),
  dueDate: z.string().datetime().optional(),
});

// 在路由处理器中进行验证
app.post('/api/tasks', async (req, res) => {
  const result = CreateTaskSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid input',
        details: result.error.flatten(),
      },
    });
  }
  // result.data 现在已经过类型检查和验证
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
    throw new ValidationError('File type not allowed');
  }
  if (file.size > MAX_SIZE) {
    throw new ValidationError('File too large (max 5MB)');
  }
  // 不要信任文件扩展名——关键场景下应检查 magic bytes
}
```

## 分类处理 npm audit 结果

并非所有审计发现都需要立即处理。使用以下决策树：

```
npm audit 报告了一个漏洞
├── 严重程度：critical 或 high
│   ├── 漏洞代码在你的应用中是否可达？
│   │   ├── 是 --> 立即修复（更新、补丁或替换依赖）
│   │   └── 否（仅开发依赖、未使用的代码路径） --> 尽快修复，但不阻塞发布
│   └── 是否有可用的修复？
│       ├── 是 --> 更新到已修补的版本
│       └── 否 --> 查找变通方案，考虑替换依赖，或加入白名单并设置审查日期
├── 严重程度：moderate
│   ├── 生产环境中可达？ --> 在下一个发布周期修复
│   └── 仅开发环境？ --> 方便时修复，纳入待办事项跟踪
└── 严重程度：low
    └── 在常规依赖更新期间跟踪并修复
```

**关键问题：**
- 漏洞函数在你的代码路径中是否真正被调用？
- 该依赖是运行时依赖还是仅开发依赖？
- 在你的部署环境下该漏洞是否可被利用（例如，纯客户端应用中的服务端漏洞）？

当你推迟修复时，记录原因并设置审查日期。

### 供应链卫生

`npm audit` 可以发现已知的 CVE；但它无法捕获恶意包或拼写错误的包（typosquat）。此外：

- **提交 lockfile** 并在 CI 中使用 `npm ci`（而非 `npm install`）安装——可复现构建，避免版本悄然漂移。
- **添加新依赖前进行审查** ——维护状况、下载量，以及它是否真正值得引入。每一个依赖都是攻击面（OWASP **A06: Vulnerable Components**，**LLM03: Supply Chain**）。
- **对不熟悉的包中的 `postinstall` 脚本保持警惕** ——它们在安装时执行任意代码。
- **注意拼写错误的包名** ——`cross-env` vs `crossenv`、`react-dom` vs `reactdom`。

## 速率限制

```typescript
import rateLimit from 'express-rate-limit';

// 通用 API 速率限制
app.use('/api/', rateLimit({
  windowMs: 15 * 60 * 1000, // 15 分钟
  max: 100,                   // 每个时间窗口 100 次请求
  standardHeaders: true,
  legacyHeaders: false,
}));

// 对认证端点设置更严格的限制
app.use('/api/auth/', rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,  // 每 15 分钟 10 次尝试
}));
```

## 密钥管理

```
.env 文件：
  ├── .env.example  → 提交到版本控制（包含占位符值的模板）
  ├── .env          → 不提交（包含真实密钥）
  └── .env.local    → 不提交（本地覆盖配置）

.gitignore 必须包含：
  .env
  .env.local
  .env.*.local
  *.pem
  *.key
```

**提交前务必检查：**
```bash
# 检查是否意外暂存了密钥
git diff --cached | grep -i "password\|secret\|api_key\|token"
```

**如果密钥曾被提交，立即轮换它。** 仅删除该行或重写历史是不够的——假设密钥在到达远程仓库的那一刻就已经泄露。先撤销并重新签发密钥，然后再从历史中清除。

## 保护 AI / LLM 功能

如果你的应用调用 LLM——聊天机器人、摘要生成器、智能体、RAG——它会继承新的攻击面。参考 [OWASP Top 10 for LLM Applications (2025)](https://genai.owasp.org/llm-top-10/) 进行映射：

- **将所有模型输出视为不可信输入（LLM05: Improper Output Handling）。** 永远不要将 LLM 输出直接传入 `eval`、SQL、shell、`innerHTML` 或文件路径。像处理原始用户输入一样对其进行验证和编码。
- **假设提示词可能被劫持（LLM01: Prompt Injection）。** 上下文窗口中的不可信文本——用户消息、抓取的网页、PDF——都可能携带指令。系统提示词不是安全边界；在代码中执行权限控制，而非在提示词中。
- **将密钥和其他用户的数据排除在提示词之外（LLM02 / LLM07）。** 上下文中的任何内容都可能被回显。不要将 API 密钥、跨租户数据或完整的系统提示词放在模型可以重复的位置。
- **约束工具和智能体的权限（LLM06: Excessive Agency）。** 将工具权限限定在最小范围，对破坏性或不可逆操作要求确认，并验证每一个工具参数。
- **限制资源消耗（LLM10: Unbounded Consumption）。** 对 token 数量、请求频率和循环/递归深度设置上限，防止精心构造的输入导致成本飙升或系统挂起。
- **隔离检索数据（LLM08: Vector and Embedding Weaknesses）。** 在 RAG 中，将向量存储视为信任边界：按租户分区嵌入数据，防止一个用户检索到其他用户的数据，并在索引前验证文档，防止被投毒的内容影响回答。

```typescript
// 错误：将模型输出作为命令或标记信任
const sql = await llm.generate(`Write SQL for: ${userQuestion}`);
await db.query(sql);                                   // 任意查询执行
container.innerHTML = await llm.reply(userMessage);   // 存储型 XSS，通过模型实现

// 正确：模型输出是数据——防御性解析，然后验证，再编码
let intent;
try {
  intent = CommandSchema.parse(JSON.parse(await llm.replyJson(userMessage)));
} catch {
  throw new ValidationError('unexpected model output'); // JSON.parse 或 schema 验证失败
}
await runAllowlistedAction(intent.action, intent.params);
container.textContent = await llm.reply(userMessage);
```

## 安全审查清单

```markdown
### 身份认证
- [ ] 密码使用 bcrypt/scrypt/argon2 进行哈希处理（salt rounds ≥ 12）
- [ ] 会话 token 设置为 httpOnly、secure、sameSite
- [ ] 登录接口有速率限制
- [ ] 密码重置 token 有过期时间

### 授权
- [ ] 每个端点都检查用户权限
- [ ] 用户只能访问自己的资源
- [ ] 管理员操作需要管理员角色验证

### 输入
- [ ] 所有用户输入在边界处进行验证
- [ ] SQL 查询使用参数化
- [ ] HTML 输出经过编码/转义
- [ ] 服务端 URL 请求使用白名单（防止 SSRF 访问内部服务）

### 数据
- [ ] 代码或版本控制中没有密钥
- [ ] 敏感字段从 API 响应中排除
- [ ] PII 在存储时加密（如适用）

### 基础设施
- [ ] 安全响应头已配置（CSP、HSTS 等）
- [ ] CORS 限制为已知来源
- [ ] 依赖已审计漏洞
- [ ] 错误消息不暴露内部信息

### 供应链
- [ ] Lockfile 已提交；CI 使用 `npm ci` 安装
- [ ] 新依赖已审查（维护状况、下载量、postinstall 脚本）

### AI / LLM（如使用）
- [ ] 模型输出视为不可信（不使用 eval/SQL/innerHTML/shell）
- [ ] 密钥和其他用户数据未放入提示词
- [ ] 工具/智能体权限已限定范围；破坏性操作需要确认
```
## 另请参阅

有关详细的安全清单和提交前验证步骤，请参见 `references/security-checklist.md`。

## 常见自我合理化

| 合理化说法 | 现实 |
|---|---|
| "这是内部工具，安全不重要" | 内部工具也会被攻破。攻击者针对最薄弱的环节。 |
| "我们以后再添加安全措施" | 安全加固的成本是从一开始就内置的 10 倍。现在就加上。 |
| "没有人会尝试利用这个漏洞" | 自动化扫描器会发现它。隐蔽式安全不是安全。 |
| "框架会处理安全" | 框架提供工具，而非保证。你仍然需要正确使用它们。 |
| "这只是一个原型" | 原型会变成生产代码。从第一天起就养成安全习惯。 |
| "这里的威胁建模过于夸张" | 五分钟的"我会如何攻击这个？"可以防止任何控制措施都无法修补的设计缺陷。 |
| "这只是 LLM 输出，不过是文本而已" | 那段"文本"可能是一条 SQL 语句、一个 script 标签或一条 shell 命令。像对待任何不可信输入一样对待它。 |

## 危险信号

- 用户输入直接传递给数据库查询、shell 命令或 HTML 渲染
- 源代码或提交历史中存在密钥
- API 端点缺少身份认证或授权检查
- 缺少 CORS 配置或使用通配符（`*`）来源
- 认证端点没有速率限制
- 堆栈跟踪或内部错误暴露给用户
- 依赖中存在已知的严重漏洞
- 服务器在没有白名单的情况下请求用户提供的 URL（SSRF）
- LLM/模型输出被传入查询、DOM、shell 或 `eval`
- 密钥、PII 或完整的系统提示词被放入 LLM 上下文窗口

## 验证

在实现安全相关代码后：

- [ ] `npm audit` 没有显示 critical 或 high 级别的漏洞
- [ ] 源代码或 git 历史中没有密钥
- [ ] 所有用户输入在系统边界处经过验证
- [ ] 每个受保护的端点都检查了身份认证和授权
- [ ] 响应中包含安全响应头（使用浏览器 DevTools 检查）
- [ ] 错误响应不暴露内部详情
- [ ] 认证端点已启用速率限制
- [ ] 服务端 URL 请求已根据白名单验证（无 SSRF）
- [ ] LLM/模型输出在使用前经过验证和编码（如有 AI 功能）
