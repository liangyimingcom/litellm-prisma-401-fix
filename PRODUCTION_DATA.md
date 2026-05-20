# 生产环境实测数据（案例参考）

> ⚠️ 本文件包含具体生产环境的配置和指标数据，仅作为排查参考。
> 你的环境数值会不同，但错误模式和修复方案相同。

---

## 环境信息

| 组件 | 版本/配置 |
|------|-----------|
| LiteLLM | ~v1.61.0（自定义镜像 `prefill-fix-v2`） |
| Prisma Python Client | 0.11.0（已归档，不再维护） |
| Prisma Query Engine | 5.22.0 |
| ECS Fargate Tasks | 16 个（4GB RAM，2 vCPU） |
| RDS | db.r7g.large，PostgreSQL 15.14，Multi-AZ |
| Redis | ElastiCache cache.r7g.large × 2（路由缓存） |
| Region | us-east-1 |
| ALB | app/litellm-stack-alb/ac78598449289944 |
| ALB idle_timeout | 4000s |
| ECS Cluster | litellm-stack-cluster |
| ECS Service | LiteLLMService |
| Log Group | /ecs/litellm-stack-litellm |
| 客户端 | OpenCode 1.14.22（ai-sdk/provider-utils 4.0.23，bun 1.3.13） |
| LLM 模型 | bedrock/us.anthropic.claude-sonnet-4-6（100% 流量） |

---

## 生产数据（2026-05-19 至 2026-05-20）

### 汇总指标

| 指标 | 数值 |
|------|------|
| engine_process_death 事件（24h） | 268 次（134 次崩溃+重连循环） |
| 请求失败（24h） | 162 次 |
| 高峰小时（CST 11:00） | 68 次 death + 60 次失败 |
| 受影响容器 | 4+/16 个 |
| ECS Memory Maximum | 44%（远低于 OOM 阈值） |
| ECS Memory Average | 36-37% |
| Prisma 重连时间 | ~100ms |
| 下次请求恢复时间 | ~3 秒 |
| 总请求量（24h） | ~19,390 |
| 失败率 | ~0.85% |
| RDS 连接数 | Avg 82，Max 111（上限 1000） |
| RDS CPU | Avg 3.7%，Max 6.5% |
| Redis CPU | Avg 0.14%，Max 0.22% |

### 按小时分布

| 时段 (UTC) | 北京时间 | engine_process_death | 请求失败 |
|-----------|---------|---------------------|----------|
| 2026-05-19 14:00 | 22:00 | 26 | 21 |
| 2026-05-19 15:00 | 23:00 | 14 | 6 |
| 2026-05-20 01:00 | 09:00 | 36 | 12 |
| 2026-05-20 02:00 | 10:00 | 38 | 6 |
| **2026-05-20 03:00** | **11:00** | **68** | **60** |
| 2026-05-20 04:00 | 12:00 | 20 | 9 |
| 2026-05-20 05:00 | 13:00 | 22 | 6 |
| 2026-05-20 06:00 | 14:00 | 40 | 36 |
| 2026-05-20 07:00 | 15:00 | 4 | 6 |

### 受影响容器列表

| Container ID | 行为 |
|:---|:---|
| 477fafa79a20 | engine_process_death + 请求失败 |
| 876704f8027e | engine_process_death + 请求失败 |
| 6edc1183d755 | engine_process_death + 请求失败 |
| 893264e5ed50 | engine_process_death + 请求失败 |

> 说明：16 个 ECS task 中 4+ 个同时受影响，证明是集群级问题而非单容器故障。

---

## 请求追踪样本

### 样本 request_id: `e9acf565-6100-4b4c-9207-f453554f0f13`

```
06:40:40.947  请求进入，pre_call_utils 处理完成
06:40:46.474  PrismaClient: find_unique for token（尝试查询 DB）
06:40:46.597  Found prisma-query-engine at PID 48778（检测到新引擎）
06:40:46.597  Prisma DB reconnect succeeded. reason=engine_process_death
06:40:46.632  返回 401 Unauthorized（当前请求已失败）
06:40:49.548  下一个请求 spend tracking 正常 ← 3秒后完全恢复
```

### 关键时间窗口

| 阶段 | 耗时 |
|------|------|
| 引擎崩溃检测 + 重连 | ~100ms |
| 重连后到当前请求失败 | ~35ms |
| 到下一个请求正常 | ~3 秒 |

---

## 穷举搜索：确认无真实认证错误

| 搜索模式 | 结果 |
|----------|------|
| `"AuthenticationError"` | 0 条 |
| `"LiteLLM_AuthenticationError"` | 0 条 |
| `"ProxyException"` | 0 条 |
| `"Invalid proxy server token"` | 0 条 |
| `"key not valid"` | 0 条 |
| `" 401 "`（宽松搜索） | 4 条 — 全部为用户 prompt 中的代码片段 |

**结论**：ECS 应用日志中零真实认证失败记录。

---

## 原始配置（修复前）

```yaml
model_list:
  - model_name: claude-sonnet-4-6
    litellm_params:
      model: bedrock/us.anthropic.claude-sonnet-4-6
      aws_region_name: us-east-1
  # ... 其他模型

litellm_settings:
  callbacks: custom_callbacks.proxy_handler_instance
  drop_params: true
  cache: true
  cache_params:
    type: redis
  request_timeout: 4000          # 67 分钟（过高）
  # 无 set_verbose、json_logs 等

router_settings:
  routing_strategy: usage-based-routing-v2
  enable_pre_call_check: true

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/DATABASE_URL
  # ❌ 缺少 allow_requests_on_db_unavailable
  # ❌ 缺少 proxy_batch_write_at
  # ❌ 缺少 disable_error_logs
```

**注意**：所有 P0/P1 缓解配置项在修复前均未设置。

---

## 排查中的 epoch 时间计算注意事项

在使用 `aws logs filter-log-events --start-time` 时需注意：

```python
# 正确的 epoch 计算方式
from datetime import datetime, timezone
epoch_ms = int(datetime(2026, 5, 19, 4, 36, 0, tzinfo=timezone.utc).timestamp() * 1000)
# → 1779165360000

# 错误：手动计算容易偏移数天
# 我们最初使用了错误的 1779681360000（差了约 6 天），导致查询结果为空
```

> 建议：始终使用 Python `datetime` 计算 epoch，不要手动换算。
