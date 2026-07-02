# Security Checklist（安全检查清单）

Web 应用安全性快速参考。配合 `security-and-hardening` skill 使用。

## 目录

- [Threat Modeling（威胁建模——从这里开始）](#threat-modeling-威胁建模从这里开始)
- [Pre-Commit Checks（提交前检查）](#pre-commit-checks-提交前检查)
- [Authentication（身份验证）](#authentication-身份验证)
- [Authorization（授权）](#authorization-授权)
- [Input Validation（输入验证）](#input-validation-输入验证)
- [Security Headers（安全头）](#security-headers-安全头)
- [CORS Configuration（CORS 配置）](#cors-configuration-cors-配置)
- [Data Protection（数据保护）](#data-protection-数据保护)
- [Dependency Security（依赖安全）](#dependency-security-依赖安全)
- [AI / LLM Security（AI / LLM 安全）](#ai--llm-security-ai--llm-安全)
- [Error Handling（错误处理）](#error-handling-错误处理)
- [OWASP Top 10 快速参考](#owasp-top-10-快速参考)
- [OWASP Top 10 for LLMs 快速参考](#owasp-top-10-for-llms-快速参考)

## Threat Modeling（威胁建模——从这里开始）

在着手实施控制措施之前，花五分钟站在攻击者的角度思考：

- [ ] 信任边界已映射（请求、上传文件、webhook、第三方 API、LLM 输出）
- [ ] 资产已命名（凭据、PII、支付数据、管理员操作、资金流转）
- [ ] 每个边界执行了 STRIDE（Spoofing 欺骗、Tampering 篡改、Repudiation 抵赖、Info disclosure 信息泄露、DoS 拒绝服务、Elevation 权限提升）
- [ ] 在每个用例旁边写出了滥用案例（"我该如何滥用这个？"）

## Pre-Commit Checks（提交前检查）

- [ ] 代码中无密钥（`git diff --cached | grep -i "password\|secret\|api_key\|token"`）
- [ ] `.gitignore` 涵盖：`.env`、`.env.local`、`*.pem`、`*.key`
- [ ] `.env.example` 使用占位值（而非真实密钥）

## Authentication（身份验证）

- [ ] 密码使用 bcrypt（≥12 轮）、scrypt 或 argon2 哈希
- [ ] Session Cookie：`httpOnly`、`secure`、`sameSite: 'lax'`
- [ ] 会话过期已配置（合理的 max-age）
- [ ] 登录端点有速率限制（每 15 分钟 ≤ 10 次尝试）
- [ ] 密码重置 Token：限时（≤ 1 小时）、一次性使用
- [ ] 多次失败后账户锁定（可选，附带通知）
- [ ] 敏感操作支持 MFA（可选但推荐）

## Authorization（授权）

- [ ] 每个受保护的端点都检查身份验证
- [ ] 每次资源访问都检查所有权/角色（防止 IDOR）
- [ ] 管理员端点需要验证管理员角色
- [ ] API Key 范围限定为最小必要权限
- [ ] JWT Token 已验证（签名、过期时间、签发者）

## Input Validation（输入验证）

- [ ] 所有用户输入在系统边界处验证（API 路由、表单处理器）
- [ ] 验证使用 allowlist（白名单）而非 denylist（黑名单）
- [ ] 字符串长度受约束（最小值/最大值）
- [ ] 数值范围已验证
- [ ] 使用合适的库验证 Email、URL 和日期格式
- [ ] 文件上传：类型受限、大小受限、内容已验证
- [ ] SQL 查询参数化（不使用字符串拼接）
- [ ] HTML 输出已编码（使用框架的自动转义）
- [ ] URL 在重定向前已验证（防止开放重定向）
- [ ] 服务端 URL 请求使用 allowlist；阻止私有/保留 IP（防止 SSRF）

## Security Headers（安全头）

```
Content-Security-Policy: default-src 'self'; script-src 'self'
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0  （已禁用，依赖 CSP）
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

## CORS Configuration（CORS 配置）

```typescript
// 严格的（推荐）
cors({
  origin: ['https://yourdomain.com', 'https://app.yourdomain.com'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
})

// 切勿在生产环境使用：
cors({ origin: '*' })  // 允许任意来源
```

## Data Protection（数据保护）

- [ ] 敏感字段不在 API 响应中暴露（`passwordHash`、`resetToken` 等）
- [ ] 敏感数据不记录到日志（密码、Token、完整信用卡号）
- [ ] PII 在静态存储中加密（若法规要求）
- [ ] 所有外部通信使用 HTTPS
- [ ] 数据库备份已加密

## Dependency Security（依赖安全）

```bash
# 审计依赖
npm audit

# 自动修复（在可能的情况下）
npm audit fix

# 检查严重漏洞
npm audit --audit-level=critical

# 保持依赖更新
npx npm-check-updates
```

**供应链安全卫生**（`npm audit` 不会捕获恶意包）：
- [ ] lockfile 已提交；CI 使用 `npm ci` 安装（而非 `npm install`）
- [ ] 新依赖已审查（维护情况、下载量、`postinstall` 脚本）
- [ ] 无拼写钓鱼包（`cross-env` vs `crossenv`、`react-dom` vs `reactdom`）

## AI / LLM Security（AI / LLM 安全）

适用于任何调用 LLM 的功能（聊天机器人、摘要器、Agent、RAG）：

- [ ] 模型输出视为不可信——绝不输入到 `eval`/SQL/shell/`innerHTML`/文件路径中
- [ ] 已假设存在 prompt injection；权限在代码中执行，而非在 system prompt 中
- [ ] 密钥、跨租户数据和完整 system prompt 不放入上下文窗口
- [ ] 工具/Agent 权限范围已限定；破坏性或不可逆操作需要确认
- [ ] 已设置 Token、速率和递归/循环限制（限制资源消耗）

## Error Handling（错误处理）

```typescript
// 生产环境：通用错误信息，不暴露内部细节
res.status(500).json({
  error: { code: 'INTERNAL_ERROR', message: 'Something went wrong' }
});

// 切勿在生产环境使用：
res.status(500).json({
  error: err.message,
  stack: err.stack,         // 暴露内部信息
  query: err.sql,           // 暴露数据库细节
});
```

## OWASP Top 10 快速参考

| # | 漏洞 | 预防措施 |
|---|---|---|
| 1 | Broken Access Control（访问控制失效） | 每个端点检查身份验证，验证所有权 |
| 2 | Cryptographic Failures（加密失败） | HTTPS、强哈希算法、代码中无密钥 |
| 3 | Injection（注入） | 参数化查询、输入验证 |
| 4 | Insecure Design（不安全的设计） | 威胁建模、spec-driven development |
| 5 | Security Misconfiguration（安全配置错误） | 安全头、最小权限、审计依赖 |
| 6 | Vulnerable Components（易受攻击的组件） | `npm audit`、保持依赖更新、最小化依赖 |
| 7 | Auth Failures（身份验证失败） | 强密码、速率限制、会话管理 |
| 8 | Data Integrity Failures（数据完整性失败） | 验证更新/依赖、签名制品 |
| 9 | Logging Failures（日志记录失败） | 记录安全事件，不记录密钥 |
| 10 | SSRF | 验证/allowlist URL、限制外发请求 |

## OWASP Top 10 for LLMs 快速参考

适用于具有 LLM 功能的应用。参见 [OWASP GenAI Security Project](https://genai.owasp.org/llm-top-10/)。

| ID | 风险 | 预防措施 |
|---|---|---|
| LLM01 | Prompt Injection（提示注入） | 不要将 system prompt 作为安全边界；在代码中执行权限控制 |
| LLM02 | Sensitive Information Disclosure（敏感信息泄露） | 密钥/PII 不放入 prompt；过滤输出 |
| LLM03 | Supply Chain（供应链） | 像审查任何依赖一样审查模型、数据集和插件 |
| LLM04 | Data and Model Poisoning（数据与模型投毒） | 使用受信任的模型来源，验证完整性；审查微调和 RAG 数据 |
| LLM05 | Improper Output Handling（不正确的输出处理） | 将模型输出视为不可信；验证、参数化、编码 |
| LLM06 | Excessive Agency（过度代理） | 限定工具权限；确认破坏性操作 |
| LLM07 | System Prompt Leakage（System Prompt 泄露） | 假设 system prompt 可能泄露；不在其中放密钥 |
| LLM08 | Vector and Embedding Weaknesses（向量与嵌入弱点） | 按租户隔离 RAG 嵌入；索引前验证文档 |
| LLM09 | Misinformation（错误信息） | 用引文支撑答案；验证关键主张；保持人工在环 |
| LLM10 | Unbounded Consumption（无限制资源消耗） | 限制 Token 数、请求速率和循环/递归深度 |
