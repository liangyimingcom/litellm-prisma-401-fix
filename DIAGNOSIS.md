# 诊断命令手册

复现排查过程，确认你的环境是否存在此问题。

## 前置条件

```bash
# 设置 AWS 凭证
export AWS_PROFILE=your-profile
export AWS_REGION=us-east-1
export LOG_GROUP=/ecs/litellm-stack-litellm  # 你的 LiteLLM 日志组
```

## 步骤 1：检查 engine_process_death 事件

```bash
# 统计最近 6 小时的 engine_process_death 事件
END=$(date +%s000)
START=$(( $(date +%s) - 21600 ))000

aws logs filter-log-events \
  --log-group-name $LOG_GROUP \
  --start-time $START --end-time $END \
  --filter-pattern 'engine_process_death' \
  --region $AWS_REGION | python3 -c "
import json, sys
data = json.load(sys.stdin)
events = data.get('events', [])
print(f'engine_process_death 事件数 (6h): {len(events)}')
if len(events) > 0:
    print('⚠️  Prisma engine 正在崩溃 — 你受此问题影响。')
else:
    print('✅ 未检测到 Prisma engine 崩溃。')
"
```

**如果计数 > 0**：你受此问题影响。继续步骤 2。

## 步骤 2：检查请求失败数

```bash
aws logs filter-log-events \
  --log-group-name $LOG_GROUP \
  --start-time $START --end-time $END \
  --filter-pattern '"request status: failure"' \
  --region $AWS_REGION | python3 -c "
import json, sys
data = json.load(sys.stdin)
events = data.get('events', [])
print(f'请求失败数 (6h): {len(events)}')
"
```

## 步骤 3：确认没有真实的认证错误

```bash
# 以下应全部返回 0
for pattern in '"AuthenticationError"' '"ProxyException"' '"Invalid proxy server token"' '"key not valid"'; do
  COUNT=$(aws logs filter-log-events \
    --log-group-name $LOG_GROUP \
    --start-time $START --end-time $END \
    --filter-pattern "$pattern" \
    --region $AWS_REGION | grep -c '"message"' || echo 0)
  echo "$pattern: $COUNT"
done
```

**预期**：全部为 0。如果有非零值，说明除了 Prisma 崩溃外，你可能还有真实的认证问题。

## 步骤 4：确认非 OOM

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name MemoryUtilization \
  --dimensions Name=ClusterName,Value=YOUR_CLUSTER \
               Name=ServiceName,Value=YOUR_SERVICE \
  --start-time $(date -u -d '-6 hours' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 3600 --statistics Average Maximum \
  --region $AWS_REGION
```

**如果 Maximum < 80%**：非 OOM。崩溃是 Prisma engine 的 bug。

## 步骤 5：追踪某个失败请求

```bash
# 找到一条最近的失败记录并获取其 request_id
aws logs filter-log-events \
  --log-group-name $LOG_GROUP \
  --start-time $START --end-time $END \
  --filter-pattern '"request status: failure"' \
  --limit 1 \
  --region $AWS_REGION | python3 -c "
import json, sys, re
data = json.load(sys.stdin)
for event in data.get('events', []):
    msg = event['message']
    match = re.search(r'request_id[\":\s]+([a-f0-9-]+)', msg)
    if match:
        print(f'Request ID: {match.group(1)}')
        print(f'Timestamp: {event[\"timestamp\"]}')
        print(f'Log stream: {event[\"logStreamName\"]}')
        break
"
```

然后追踪完整的请求生命周期：

```bash
REQUEST_ID="粘贴上面获取的 request-id"

aws logs filter-log-events \
  --log-group-name $LOG_GROUP \
  --start-time $START --end-time $END \
  --filter-pattern "\"$REQUEST_ID\"" \
  --region $AWS_REGION | python3 -c "
import json, sys
data = json.load(sys.stdin)
for event in sorted(data.get('events', []), key=lambda e: e['timestamp']):
    print(event['message'][:200])
    print()
"
```

**关注**：在 401 同一时间窗口是否有 `engine_process_death` 或 `reconnect`。

## 步骤 6：检查 Prisma 版本

```bash
aws logs filter-log-events \
  --log-group-name $LOG_GROUP \
  --filter-pattern '"prisma-query-engine"' \
  --limit 3 \
  --region $AWS_REGION
```

如果你看到 Prisma Query Engine 5.x 配合 prisma-client-py 0.11.0，说明你正在使用已废弃、不再维护的版本。
