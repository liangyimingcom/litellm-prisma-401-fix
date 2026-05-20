# LiteLLM Proxy: Prisma `engine_process_death` 导致虚假 HTTP 401 — 根因分析与修复方案

## 概述

LiteLLM Proxy (v1.61+) 在其内部数据库引擎（Prisma Query Engine）间歇性崩溃时，会向客户端返回 **HTTP 401 Unauthorized**。这**不是**真正的认证失败 — 而是一个被错误分类的内部错误。像 [OpenCode](https://opencode.ai) 这样的客户端不会自动重试 401 错误，导致用户看到误导性的"认证失败"提示。

**一行修复**（添加到 `config.yaml`）：

```yaml
general_settings:
  allow_requests_on_db_unavailable: True
```

这告诉 LiteLLM 在 Prisma 不可用时绕过数据库检查，允许请求正常发送到 LLM 后端。

---

## 目录

1. [问题总览](#问题总览)
2. [故障现象](#故障现象)
3. [根因分析](#根因分析)
4. [排查过程](#排查过程)
5. [关键证据](#关键证据)
6. [修复方案](#修复方案)
7. [客户端影响（OpenCode / ai-sdk）](#客户端影响opencode--ai-sdk)
8. [修复验证](#修复验证)
9. [参考链接](#参考链接)

---

## 问题总览

### 问题描述

当 LiteLLM Proxy 使用 PostgreSQL 数据库后端（用于 spend tracking、key 管理等）时，Prisma Query Engine 进程会间歇性崩溃（`engine_process_death`）。在约 100ms 的重连窗口期内，正在处理的请求会失败。LiteLLM 错误地返回 **HTTP 401 Unauthorized** 而非预期的 **503 Service Unavailable**。

### 为什么重要

- **客户端不会自动重试 401** — HTTP 401 被归类为客户端侧认证错误。SDK（Vercel AI SDK、OpenAI SDK）仅自动重试 5xx 错误。
- **用户看到"认证失败"** — 误导性错误信息；用户浪费时间检查 API key。
- **问题持续存在** — Prisma engine 崩溃持续发生（我们的生产环境每天 100+ 次）。

### 解决方案

| 优先级 | 操作 | 效果 |
|--------|------|------|
| **P0** | `allow_requests_on_db_unavailable: True` | 请求绕过 DB → **401 消除** |
| **P0** | `proxy_batch_write_at: 60` | 减少 Prisma 调用频率约 60 倍 |
| P1 | `disable_error_logs: True` | 进一步减少 DB 写入（错误仍通过 S3/callback 记录） |
| P1 | `request_timeout: 600` | 防止极端长尾请求（默认 6000s） |

---

## 故障现象

### 用户侧看到的

```
Error: Unauthorized: {"error":{"message":"Authentication Error...","code":401}}
```

在 OpenCode 终端中：
```
⚠ Authentication failed. Check your API key.
```

### 运维侧日志

```
WARNING  - Attempting Prisma DB reconnect. reason=engine_process_death
INFO     - Prisma DB reconnect succeeded. reason=engine_process_death
INFO     - Logged request status: failure
INFO     - "POST /v1/messages HTTP/1.1" 401 Unauthorized
```

### 关键特征

| 特征 | 观察值 |
|------|--------|
| 错误码 | HTTP 401（不是 500/502/503） |
| 频率 | 每 24h 40-160+ 次失败（我们的生产环境：162 次/24h） |
| 分布 | 多个 ECS 容器同时受影响 |
| 恢复 | 下一次请求成功（约 3s 后） |
| 认证状态 | API key 有效（内存缓存命中已确认） |
| Bedrock 调用 | 从未到达 Bedrock（prompt_tokens=0, spend=0） |
| 内存 | 非 OOM（我们的最高值：44%，4GB 中） |

---

## 根因分析

### 因果链

```
┌─────────────────────────────────────────────────────────────────┐
│  请求流程（当 Prisma engine 崩溃时）：                              │
│                                                                 │
│  1. 客户端发送 POST /v1/messages，携带有效 API key                │
│  2. ✅ LiteLLM 验证 API key（内存缓存命中）                       │
│  3. ✅ Router 选择 deployment（bedrock/claude-sonnet-4-6）        │
│  4. ❌ LiteLLM 查询 Prisma 获取 key 配额/spend tracking           │
│     → Prisma Query Engine 进程此时已死亡                          │
│     → 抛出连接异常                                               │
│  5. ❌ LiteLLM 异常处理器将此归类为认证失败                        │
│  6. ❌ 返回 HTTP 401 给客户端                                     │
│                                                                 │
│  同时：Prisma 在 ~100ms 内自动重连                               │
│  下次请求：约 3s 后正常处理                                       │
└─────────────────────────────────────────────────────────────────┘
```

### 为什么 Prisma Engine 会崩溃

1. **prisma-client-py 已被废弃** — Python Prisma 客户端（v0.11.0）于 [2025-04-15 归档](https://github.com/RobertCraigie/prisma-client-py/issues/1073)。维护者停止开发，因为 Prisma 正在将核心从 Rust 重写为 TypeScript。

2. **LiteLLM 仍然依赖它** — 截至 LiteLLM v1.86.0，`pyproject.toml` 仍指定 `prisma==0.11.0`。此问题在 [BerriAI/litellm#9753](https://github.com/BerriAI/litellm/issues/9753) 中跟踪。

3. **Query Engine 是独立的 Rust 二进制进程** — 作为子进程运行。当它 panic 或 segfault 时，Python 客户端检测到 `engine_process_death` 并重连。

4. **生产环境崩溃频率** — 我们的数据显示 **268 次/24 小时**（134 次崩溃+重连循环），分布在 16 个 ECS 容器中的 4+ 个。

### 为什么是 401 而不是 503

LiteLLM 的错误处理逻辑将"无法在 DB 中验证 key 配额"与"认证失败"混为一谈。当 `allow_requests_on_db_unavailable` 未设置时（默认），auth/quota 检查路径中的任何 Prisma 异常都会导致 401 响应 — 即使 API key 已通过内存验证。

---

## 排查过程

### 步骤 1：识别错误模式

CloudWatch Logs 查询：
```
filter @message like /engine_process_death/
| stats count(*) as death_count by bin(1h)
```

结果：24h 内 268 个事件，集中在工作时间。

### 步骤 2：关联请求失败

```
filter @message like /request status: failure/
| stats count(*) as failures by bin(1h)
```

结果：162 次失败 — 与 engine_process_death 事件**高度正相关**。

### 步骤 3：追踪单个请求

使用 request_id `e9acf565-6100-4b4c-9207-f453554f0f13`：

```
06:40:46.474  PrismaClient: find_unique for token（查询 DB）
06:40:46.597  Found prisma-query-engine at PID 48778（检测到新引擎）
06:40:46.597  Prisma DB reconnect succeeded. reason=engine_process_death
06:40:46.632  "POST /v1/messages HTTP/1.1" 401 Unauthorized
```

**关键发现**：重连成功了，但当前请求已进入失败路径 — 这是一个竞态条件。

### 步骤 4：排除真实认证错误

穷举搜索所有认证错误格式：
```
"AuthenticationError"           → 0 条
"LiteLLM_AuthenticationError"   → 0 条
"ProxyException"                → 0 条
"Invalid proxy server token"    → 0 条
"key not valid"                 → 0 条
```

**结论**：零真实认证失败。所有"401"错误均由 Prisma 崩溃引起。

### 步骤 5：确认非 OOM

```bash
aws cloudwatch get-metric-statistics --namespace AWS/ECS \
  --metric-name MemoryUtilization \
  --dimensions Name=ClusterName,Value=litellm-stack-cluster \
                Name=ServiceName,Value=LiteLLMService \
  --statistics Average Maximum
```

结果：Average 36-37%，Maximum 44%。**非 OOM**。

---

## 关键证据

我们在生产环境中对此问题进行了完整的端到端验证。关键数据摘要：

- **24h 内 268 次 engine_process_death → 162 次请求失败**（失败率 ~0.85%）
- 高峰时段单小时 68 次 engine 崩溃 + 60 次请求失败
- 多个容器同时受影响（集群级问题，非单点故障）
- 内存使用率最高 44%（**非 OOM**）
- Prisma 重连 ~100ms，下次请求 ~3s 恢复
- 穷举搜索 `AuthenticationError` / `ProxyException` 等 → **全部为 0**（零真实认证失败）

> 📋 完整的生产数据（按小时分布、容器列表、请求追踪样本、原始配置等）见 [PRODUCTION_DATA.md](./PRODUCTION_DATA.md)

---

## 修复方案

### 最小修复（P0 — 立即应用）

在 `config.yaml` 的 `general_settings` 下添加两行：

```yaml
general_settings:
  allow_requests_on_db_unavailable: True  # Prisma 崩溃时绕过 DB 检查
  proxy_batch_write_at: 60                # 每 60s 批量写入（减少 Prisma 调用）
```

**效果**：
- `allow_requests_on_db_unavailable: True` — Prisma 不可用时，请求跳过 DB 检查直接发送到 LLM 后端。Spend tracking 暂时跳过（Prisma 恢复后继续）。
- `proxy_batch_write_at: 60` — spend 写入频率从"每个请求"改为"每 60 秒"，减少 Prisma 交互约 60 倍。

### 完整推荐配置

```yaml
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/DATABASE_URL
  allow_requests_on_db_unavailable: True    # P0: DB 故障时绕过
  proxy_batch_write_at: 60                   # P0: 批量写入
  # disable_error_logs: True                 # P1: 停止错误日志写 DB
  # database_connection_pool_limit: 10       # P1: 显式连接池大小

litellm_settings:
  # request_timeout: 600                     # P1: 从默认 6000s 降低
  # set_verbose: False                       # P1: 禁用 debug 日志
  # json_logs: true                          # P1: 结构化日志
```

### 环境变量

```bash
export LITELLM_LOG=ERROR          # 减少日志量（默认输出所有级别）
export LITELLM_MODE=PRODUCTION    # 禁用 load_dotenv()
```

### 部署步骤

```bash
# 1. 备份当前配置
aws s3 cp s3://your-config-bucket/config.yaml \
  s3://your-config-bucket/config.yaml.bak

# 2. 上传新配置
aws s3 cp ./config.yaml s3://your-config-bucket/config.yaml

# 3. 强制新部署
aws ecs update-service --cluster your-cluster \
  --service YourLiteLLMService --force-new-deployment

# 4. 验证（部署完成后）
aws logs filter-log-events --log-group-name /ecs/litellm \
  --filter-pattern 'engine_process_death' \
  --start-time $(date -d '-1 hour' +%s000)
```

---

## 客户端影响（OpenCode / ai-sdk）

### 为什么 OpenCode 不自动重试

OpenCode 使用 `@ai-sdk/openai-compatible`（Vercel AI SDK），其重试行为：

| HTTP 状态码 | SDK 行为 | 重试？ |
|------------|----------|--------|
| 429 | Rate limited | ✅ 自动重试（exponential backoff） |
| 500, 502, 503 | 服务端错误 | ✅ 自动重试 |
| **401** | 认证失败 | ❌ **立即报错，不重试** |
| 400 | 请求格式错误 | ❌ 不重试 |

由于 LiteLLM 返回 401（而非 503），SDK 直接向用户报错而不重试。**如果 LiteLLM 返回 503，SDK 会自动重试，用户完全无感知。**

### 客户端侧临时方案

如果暂时无法修改服务端配置，用户可以简单地**手动重试** — 下次请求约 3 秒后即成功。

OpenCode `.opencode.json` 推荐配置：

```json
{
  "providers": {
    "litellm": {
      "npm": "@ai-sdk/openai-compatible",
      "options": {
        "baseURL": "https://your-litellm-proxy/v1",
        "timeout": 600000
      }
    }
  }
}
```

### 最佳实践决策树

```
opencode 收到 401 错误
    │
    ├─ 是否频繁出现（>5次/小时）？
    │   ├─ 是 → 服务端问题，管理员需设置 allow_requests_on_db_unavailable: True
    │   └─ 否 → 偶发瞬时故障，直接重试即可
    │
    ├─ 重试后是否成功？
    │   ├─ 是 → Prisma 瞬时崩溃（正常恢复），无需其他操作
    │   └─ 否 → 检查 API Key 是否真的过期或被删除
    │
    └─ 长期解决方案：
        ① P0: 服务端 allow_requests_on_db_unavailable: True（消除问题）
        ② P1: LiteLLM 修正错误码（401 → 503，使 ai-sdk 自动重试）
        ③ P2: opencode 侧增加对 401 的有限重试逻辑
```

---

## 修复验证

应用修复后，使用以下命令验证：

```bash
# 检查 engine_process_death 是否仍发生（会有，但请求应该成功）
aws logs filter-log-events --log-group-name /ecs/litellm \
  --filter-pattern 'engine_process_death' \
  --start-time $(date -d '-1 hour' +%s000) | grep -c message

# 检查 401 错误（修复后应为 0）
aws logs filter-log-events --log-group-name /ecs/litellm \
  --filter-pattern '"request status: failure"' \
  --start-time $(date -d '-1 hour' +%s000) | grep -c message
```

**预期结果**：
- `engine_process_death` 可能仍会出现（Prisma engine 本身仍不稳定）
- 但**请求失败和 401 错误应降至接近零**

---

## 参考链接

| 资源 | 链接 |
|------|------|
| **prisma-client-py 废弃公告** | https://github.com/RobertCraigie/prisma-client-py/issues/1073 |
| **LiteLLM Prisma 废弃讨论** | https://github.com/BerriAI/litellm/issues/9753 |
| **LiteLLM 生产最佳实践** | http://docs.litellm.ai/docs/proxy/prod |
| **LiteLLM 重试与降级** | https://docs.litellm.ai/docs/completion/reliable_completions |
| **Prisma Query Engine Panic** | https://github.com/prisma/prisma/issues/2754 |
| **Prisma Engine 连接泄露** | https://github.com/prisma/prisma/issues/7020 |
| **Prisma ARM64 Segfault** | https://github.com/prisma/prisma/issues/18510 |
| **OpenCode + LiteLLM 中途停止** | https://github.com/anomalyco/opencode/issues/3365 |
| **OpenCode subagent 挂起（无重试）** | https://github.com/anomalyco/opencode/issues/11865 |
| **Vercel AI SDK 错误处理** | https://sdk.vercel.ai/docs/ai-sdk-ui/error-handling |
| **OpenCode 配置文档** | https://opencode-ai-opencode.mintlify.app/core-concepts/configuration |
| **LiteLLM 安全更新 (2026-03)** | https://docs.litellm.ai/blog/security-update-march-2026 |

---

## 环境信息

本问题在以下环境中验证：LiteLLM v1.61+ 搭配 Prisma Python Client 0.11.0（已废弃），部署于 ECS Fargate（PostgreSQL + Redis），客户端为 OpenCode 1.14.22。

> 📋 完整环境清单（实例规格、资源标识、网络配置等）见 [PRODUCTION_DATA.md](./PRODUCTION_DATA.md)

---

## License

本分析和修复文档采用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 许可共享。
