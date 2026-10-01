# 安全 — 密钥、API 密钥轮换、审计日志、护栏

> 中文译文。代码、命令和标识符保持英文。若与原文不一致，以同目录 docs/en.md 为准。

> 通过集中式保险库（HashiCorp Vault、AWS Secrets Manager、Azure Key Vault）消除密钥蔓延。绝不要把凭据存在配置文件、进版本控制的 env 文件、电子表格里。用 IAM 角色而不是静态密钥；CI/CD 用 OIDC。AI 网关模式是 2026 年的解法：应用 → 网关 → 模型提供方，网关在运行时从保险库拉取凭据。在保险库里轮换，所有应用在几分钟内跟上 — 不用重新部署，不用在 Slack 里问“谁有新密钥”。轮换策略 ≤90 天；每次提交用 TruffleHog / GitGuardian / Gitleaks 扫描。零信任：MFA、SSO、RBAC/ABAC、短生命周期令牌、设备态势。PII 清洗用实体识别在转发前掩码 PHI/PII；一致令牌化（Mesh 方法）把敏感值映射到稳定占位符，使 LLM 保留代码/关系语义。网络出站：LLM 服务放在专用 VPC/VNet 子网，只把 `api.openai.com`、`api.anthropic.com` 等列入白名单；阻断所有其他出站。2026 年的事件驱动因素：Vercel 供应链攻击通过被攻陷的 CI/CD 凭据，把环境变量从数千个客户部署中外泄出去。

**Type:** 理解
**Languages:** Python（标准库，玩具 PII 清洗器 + 审计日志写入器）
**Prerequisites:** Phase 17 · 19（AI 网关）、Phase 17 · 13（可观测性）
**Time:** ~60 分钟

## 学习目标

- 枚举四种密钥管理反模式（版本控制里的配置文件、硬编码环境变量、电子表格、静态密钥），并说出它们的替代。
- 解释 AI 网关从保险库拉取这一模式，作为 2026 年的生产标准。
- 实现一个带一致令牌化的 PII 清洗器（相同的值 → 相同的占位符），使语义得以保留。
- 说出 2026 年 Vercel 供应链事件，以及它关于 CI/CD 凭据卫生教了什么。

## 问题

一个实习生提交了带 API 密钥的 `.env`。他们很快删掉了它。密钥已经在 git 历史里 — GitGuardian 扫描抓住了它，你的轮换流程是“在 Slack 里通知团队，更新 40 个配置文件，重新部署所有服务。”8 小时后，一半服务已经上线，另一半还在等部署窗口。

另外，用户提示词里有 “My SSN is 123-45-6789.”。提示词去了 OpenAI。你有 BAA（业务伙伴协议），但内部政策是转发前掩码 PII。你没有做。

另外，你的 EKS 集群里的 LLM Pod 可以到达任何互联网主机。有人通过对攻击者控制域名的 DNS 查询把数据外泄出去。没有任何东西拦住它。

LLM 服务的安全必须处理全部三条向量。保险库支撑的凭据。PII 清洗。网络出站过滤。审计日志。

## 概念

### 集中式保险库 + IAM 角色拉取

**保险库**：HashiCorp Vault、AWS Secrets Manager、Azure Key Vault、GCP Secret Manager。一个事实来源。

**IAM 角色**：应用/网关用它的 IAM 身份认证，而不是静态密钥。保险库在令牌的生命周期内返回密钥。

**AI 网关模式**：网关在请求时从保险库拉取 `OPENAI_API_KEY`。在保险库里轮换；下一个请求拿到新密钥。不用重新部署。

### 轮换策略 ≤ 90 天

所有 API 密钥、保险库根令牌、CI/CD 凭据。尽可能自动轮换。手动轮换要记录并跟踪。

### 密钥扫描

- **TruffleHog** — 对提交做正则 + 熵。
- **GitGuardian** — 商业产品，准确率高。
- **Gitleaks** — 开源，在 CI 中运行。

每次提交都跑。如果检测到新密钥就阻断 PR。

### 零信任态势

- 所有账户要求 MFA。
- 通过 SAML/OIDC 做 SSO。
- 用 RBAC（基于角色）或 ABAC（基于属性）做细粒度访问。
- 短生命周期令牌（以小时计，不是以天计）。
- 设备态势 — 只有带磁盘加密的公司设备。

### PII / PHI 清洗

在提示词离开你的基础设施之前：

1. 实体识别（spaCy NER、Presidio、商业产品）。
2. 掩码匹配到的实体：`"My SSN is 123-45-6789"` → `"My SSN is [SSN_TOKEN_A3F]"`。
3. 一致令牌化（Mesh 方法）：相同的值映射到相同的占位符，使 LLM 保留关系。
4. 可选的反向映射，用于 LLM 响应。

静态正则过滤器抓住基本模式；NER 抓住更多。两者都用。

### 输入 + 输出护栏

输入：阻断已知越狱、禁止主题；按用户做速率限制。

输出：用正则清洗泄漏的密钥（API 密钥模式、拒绝语境中的邮箱模式），用分类器检查策略违规。

### 网络出站白名单

LLM 服务放在专用子网：
- 白名单：`api.openai.com`、`api.anthropic.com`、向量数据库端点、保险库端点。
- 其他一切：丢弃。
- DNS 走只允许白名单的解析器（避免 DNS 隧道外泄）。

### 审计日志

每一次 LLM 调用的不可变日志，包含：
- 时间戳。
- 用户 / 租户。
- 提示词哈希（为隐私，不是原始提示词）。
- 模型 + 版本。
- token 计数。
- 成本。
- 响应哈希。
- 任何护栏触发。

按监管要求留存（SOC 2 为 1 年，HIPAA 为 6 年）。

### 2026 年 Vercel 事件

供应链攻击：被攻陷的 CI/CD 凭据把环境变量从数千个客户部署中外泄出去。教训：CI/CD 凭据等同于生产。存在保险库里。范围收窄。积极轮换。

### 你应该记住的数字

- 轮换策略：≤ 90 天。
- 每次提交扫描：TruffleHog / GitGuardian / Gitleaks。
- Vercel 2026：CI/CD 凭据被攻陷 → 数千个客户环境变量泄漏。
- 审计日志留存：SOC 2 = 1 年，HIPAA = 6 年。

```figure
i4-vault-rotation
```

## 使用

`code/main.py` 实现一个带一致令牌化的玩具 PII 清洗器，以及一份只追加的审计日志。

## 交付

本课产出 `outputs/skill-llm-security-plan.md`。给定监管范围和当前状态，规划保险库迁移、清洗器、出站、审计日志。

## 练习

1. 运行 `code/main.py`。发送两条引用同一个 SSN 的提示词。确认两者得到同一个占位符。
2. 为一个在 EKS 上跑 vLLM、调用 OpenAI + Anthropic + Weaviate 的部署设计网络出站策略。
3. 你在 git 历史里发现一把密钥（已有 2 年）。正确的响应是轮换密钥、清洗历史，还是两者都做？给出理由。
4. 你的审计日志每天增长 10 GB。设计留存分层（热 30 天、温 12 个月、冷 6 年）。
5. 论证反向令牌化（把真实值代回 LLM 响应）是否值得其复杂度，相对于让占位符保持可见。

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| 保险库 | “密钥存储” | 集中式凭据管理服务 |
| IAM 角色 | “基于身份的认证” | 由应用承担的角色；返回短生命周期凭据 |
| 用于 CI/CD 的 OIDC | “云签发的令牌” | CI 里没有静态密钥 — 通过 OIDC 证明身份 |
| TruffleHog / GitGuardian / Gitleaks | “密钥扫描器” | 提交时的密钥检测 |
| RBAC / ABAC | “访问控制” | 基于角色相对基于属性 |
| PII 清洗 | “数据掩码” | 移除或令牌化敏感实体 |
| 一致令牌化 | “稳定占位符” | 相同的值每次都变成相同的令牌 |
| Mesh 方法 | “Mesh 令牌化” | 保留语义的令牌化模式 |
| 出站白名单 | “出站允许列表” | 只有被允许的域名可达 |
| 审计日志 | “不可变历史” | 用于合规的只追加记录 |

## 延伸阅读

- [Doppler — Advanced LLM Security](https://www.doppler.com/blog/advanced-llm-security)
- [Portkey — Manage LLM API keys with secret references](https://portkey.ai/blog/secret-references-ai-api-key-management/)
- [Datadog — LLM Guardrails Best Practices](https://www.datadoghq.com/blog/llm-guardrails-best-practices/)
- [JumpServer — Secrets Management Best Practices 2026](https://www.jumpserver.com/blog/secret-management-best-practices-2026)
- [Microsoft Presidio](https://github.com/microsoft/presidio) — PII 检测与匿名化。
- [HashiCorp Vault 文档](https://developer.hashicorp.com/vault/docs)
