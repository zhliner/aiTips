# 安全检查清单（Security Checklist）

Web 应用安全快速参考。配合 `security-and-harden` skill 使用。

## 目录

- [威胁建模（从这里开始）](#威胁建模从这里开始)
- [提交前检查](#提交前检查)
- [身份认证](#身份认证)
- [授权](#授权)
- [输入验证](#输入验证)
- [安全响应头](#安全响应头)
- [CORS 配置](#cors-配置)
- [数据保护](#数据保护)
- [依赖安全](#依赖安全)
- [AI / LLM 安全](#ai--llm-安全)
- [错误处理](#错误处理)
- [OWASP Top 10 快速参考](#owasp-top-10-快速参考)
- [OWASP LLM Top 10 快速参考](#owasp-llm-top-10-快速参考)

## 威胁建模（从这里开始）

在着手实施安全措施之前，先花五分钟以攻击者的视角思考：

- [ ] 已绘制信任边界（请求、上传、webhook、第三方 API、LLM 输出）
- [ ] 已列出资产（凭证、PII、支付数据、管理员操作、资金流转）
- [ ] 已针对每个边界执行 STRIDE 分析（Spoofing、Tampering、Repudiation、Info disclosure、DoS、Elevation）
- [ ] 已在使用场景旁编写滥用场景（"我会如何 misuse 这个功能？"）

## 提交前检查

- [ ] 代码中无密钥（`git diff --cached | grep -i "password\|secret\|api_key\|token"`）
- [ ] `.gitignore` 覆盖：`.env`、`.env.local`、`*.pem`、`*.key`
- [ ] `.env.example` 使用占位符值（而非真实密钥）

## 身份认证

- [ ] 密码使用 bcrypt（≥12 轮）、scrypt 或 argon2 进行哈希处理
- [ ] Session cookie：`httpOnly`、`secure`、`sameSite: 'lax'`
- [ ] 已配置 session 过期时间（合理的 max-age）
- [ ] 登录端点已设置速率限制（每 15 分钟 ≤10 次尝试）
- [ ] 密码重置令牌：限时（≤1 小时）、一次性使用
- [ ] 多次失败后锁定账户（可选，需附带通知）
- [ ] 敏感操作支持 MFA（可选但推荐）

## 授权

- [ ] 每个受保护的端点都检查身份认证
- [ ] 每次资源访问都检查所有权/角色（防止 IDOR）
- [ ] 管理员端点要求管理员角色验证
- [ ] API 密钥的权限范围限制在最小必要权限
- [ ] JWT 令牌经过验证（签名、过期时间、签发者）

## 输入验证

- [ ] 所有用户输入在系统边界处进行验证（API 路由、表单处理器）
- [ ] 验证使用白名单（而非黑名单）
- [ ] 字符串长度受限（最小值/最大值）
- [ ] 数值范围已验证
- [ ] 邮箱、URL 和日期格式使用合适的库进行验证
- [ ] 文件上传：类型受限、大小受限、内容已验证
- [ ] SQL 查询使用参数化（不使用字符串拼接）
- [ ] HTML 输出已编码（使用框架的自动转义功能）
- [ ] 重定向前验证 URL（防止 open redirect）
- [ ] 服务端 URL 请求使用白名单；屏蔽私有/保留 IP（防止 SSRF）

## 安全响应头

```
Content-Security-Policy: default-src 'self'; script-src 'self'
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0  （已禁用，依赖 CSP）
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

## CORS 配置

```typescript
// 限制性配置（推荐）
cors({
  origin: ['https://yourdomain.com', 'https://app.yourdomain.com'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
})

// 切勿在生产环境中使用：
cors({ origin: '*' })  // 允许任意来源
```

## 数据保护

- [ ] 敏感字段已从 API 响应中排除（`passwordHash`、`resetToken` 等）
- [ ] 敏感数据未记录到日志（密码、令牌、完整信用卡号）
- [ ] PII 在静态存储时已加密（如法规要求）
- [ ] 所有外部通信使用 HTTPS
- [ ] 数据库备份已加密

## 依赖安全

```bash
# 审计依赖
npm audit

# 尽可能自动修复
npm audit fix

# 检查严重漏洞
npm audit --audit-level=critical

# 保持依赖更新
npx npm-check-updates
```

**供应链卫生**（`npm audit` 无法检测恶意包）：
- [ ] 已提交 lockfile；CI 使用 `npm ci` 安装（而非 `npm install`）
- [ ] 新依赖经过审查（维护状况、下载量、`postinstall` 脚本）
- [ ] 无 typosquat（`cross-env` vs `crossenv`、`react-dom` vs `reactdom`）

## AI / LLM 安全

对于任何调用 LLM 的功能（聊天机器人、摘要工具、智能体、RAG）：

- [ ] 模型输出视为不可信 — 绝不传入 `eval`/SQL/shell/`innerHTML`/文件路径
- [ ] 假定存在 prompt injection；权限在代码中强制执行，而非在 system prompt 中
- [ ] 密钥、跨租户数据和完整 system prompt 不进入上下文窗口
- [ ] 工具/智能体权限范围受限；破坏性或不可逆操作需确认
- [ ] 已设置 token、速率和递归/循环限制（约束消耗量）

## 错误处理

```typescript
// 生产环境：通用错误，不暴露内部信息
res.status(500).json({
  error: { code: 'INTERNAL_ERROR', message: 'Something went wrong' }
});

// 切勿在生产环境中使用：
res.status(500).json({
  error: err.message,
  stack: err.stack,         // 暴露内部信息
  query: err.sql,           // 暴露数据库细节
});
```

## OWASP Top 10 快速参考

| # | 漏洞 | 预防措施 |
|---|---|---|
| 1 | Broken Access Control | 每个端点进行权限检查，验证所有权 |
| 2 | Cryptographic Failures | HTTPS、强哈希、代码中无密钥 |
| 3 | Injection | 参数化查询、输入验证 |
| 4 | Insecure Design | 威胁建模、规格驱动开发 |
| 5 | Security Misconfiguration | 安全响应头、最小权限、审计依赖 |
| 6 | Vulnerable Components | `npm audit`、保持依赖更新、最小化依赖 |
| 7 | Auth Failures | 强密码、速率限制、session 管理 |
| 8 | Data Integrity Failures | 验证更新/依赖、签名制品 |
| 9 | Logging Failures | 记录安全事件，不记录密钥 |
| 10 | SSRF | 验证/白名单 URL、限制出站请求 |

## OWASP LLM Top 10 快速参考

适用于包含 LLM 功能的应用。参见 [OWASP GenAI Security Project](https://genai.owasp.org/llm-top-10/)。

| ID | 风险 | 预防措施 |
|---|---|---|
| LLM01 | Prompt Injection | 不要将 system prompt 视为安全边界；在代码中强制执行权限 |
| LLM02 | Sensitive Information Disclosure | 密钥/PII 不进入 prompt；过滤输出 |
| LLM03 | Supply Chain | 像审查其他依赖一样审查模型、数据集和插件 |
| LLM04 | Data and Model Poisoning | 使用可信的模型来源，验证完整性；审查 fine-tuning 和 RAG 数据 |
| LLM05 | Improper Output Handling | 将模型输出视为不可信；验证、参数化、编码 |
| LLM06 | Excessive Agency | 限制工具权限；确认破坏性操作 |
| LLM07 | System Prompt Leakage | 假定 system prompt 可能泄露；其中不放任何密钥 |
| LLM08 | Vector and Embedding Weaknesses | 按租户隔离 RAG embedding；索引前验证文档 |
| LLM09 | Misinformation | 用引用支撑回答；验证关键声明；保持人在回路中 |
| LLM10 | Unbounded Consumption | 限制 token 数、请求速率和循环/递归深度 |
