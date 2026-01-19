---
name: apm-alert-investigation
description: This skill should be used when the user asks to "investigate APM alert", "debug application performance", "check service latency", "analyze error rates", mentions application monitoring issues, JVM problems, response time degradation, error spikes, garbage collection issues, or receives alerts about application performance including HTTP errors, timeout rates, or thread pool exhaustion.
version: 2.0.0
---

# Application/APM Alert Investigation Skill v2.0

## Overview

Comprehensive protocol for investigating application performance alerts with JVM diagnostics, error analysis, and dependency correlation for Spring Boot services using iZeus APM platform.

## Quick Reference

| Data Source | UID/Server | Purpose |
|-------------|------------|---------|
| Prometheus | `df8o21agxtkw0d` (UMBQuerier-Luckin) | JVM & app metrics |
| iZeus APM | Prometheus integration | APM-specific metrics |
| CloudWatch | Logs & custom metrics | AWS-native data |
| DevOps DB | `aws-luckyus-devops-rw` | Service topology |

## Alert Categories

| Category | Trigger Conditions | Priority |
|----------|-------------------|----------|
| JVM CPU | > 80% sustained | L1/L2 |
| JVM Memory | Heap > 90% OR OOM | L0/L1 |
| GC Pressure | GC pause > 1s OR overhead > 10% | L0/L1 |
| Response Time | p99 > SLA threshold | L0/L1 |
| Error Rate | > 1% of requests | L0/L1 |
| Thread Pool | Exhausted or rejected | L0/L1 |
| HTTP Errors | 5xx rate spike | L0/L1 |

## Investigation Protocol

### Phase 1: Alert Validation (Max 30s)

Extract from alert payload:
- `service_name`, `instance_id`, `pod_name`
- Alert type, threshold, current value
- Duration and trend direction

Classify service priority:
- **L0**: payment-service, order-service, auth-service (5min SLA)
- **L1**: inventory-service, user-service, gateway (15min SLA)
- **L2**: notification-service, analytics-service (30min SLA)

### Phase 2: JVM Health Assessment (Parallel)

**CRITICAL**: Execute these Prometheus queries in parallel:

```promql
# Memory (parallel)
jvm_memory_used_bytes{service="$SERVICE",area="heap"}
jvm_memory_max_bytes{service="$SERVICE",area="heap"}
jvm_memory_used_bytes{service="$SERVICE",area="nonheap"}

# GC (parallel)
rate(jvm_gc_pause_seconds_sum{service="$SERVICE"}[5m])
rate(jvm_gc_pause_seconds_count{service="$SERVICE"}[5m])
jvm_gc_memory_promoted_bytes_total{service="$SERVICE"}

# CPU & Threads (parallel)
process_cpu_usage{service="$SERVICE"}
jvm_threads_live_threads{service="$SERVICE"}
jvm_threads_daemon_threads{service="$SERVICE"}
jvm_threads_peak_threads{service="$SERVICE"}
```

### Phase 3: Application Performance Analysis

**Response Time Metrics**:
```promql
# HTTP latency percentiles
histogram_quantile(0.50, rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m]))
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m]))
histogram_quantile(0.99, rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m]))

# Request rate
rate(http_server_requests_seconds_count{service="$SERVICE"}[5m])
```

**Error Rate Analysis**:
```promql
# Error rate by status code
sum(rate(http_server_requests_seconds_count{service="$SERVICE",status=~"5.."}[5m])) /
sum(rate(http_server_requests_seconds_count{service="$SERVICE"}[5m]))

# Error breakdown
sum by(status,uri)(rate(http_server_requests_seconds_count{service="$SERVICE",status=~"[45].."}[5m]))
```

### Phase 4: iZeus APM Metrics

```promql
# Business transaction metrics
izeus_transaction_duration_seconds{service="$SERVICE"}
izeus_transaction_count{service="$SERVICE"}
izeus_error_count{service="$SERVICE"}

# Strategy/circuit breaker status
izeus_strategy_status{service="$SERVICE"}
izeus_circuit_breaker_state{service="$SERVICE"}
```

### Phase 5: Endpoint-Level Analysis

**Top Slow Endpoints**:
```promql
topk(10,
  histogram_quantile(0.99, rate(http_server_requests_seconds_bucket{service="$SERVICE"}[5m]))
)
```

**Top Error Endpoints**:
```promql
topk(10,
  sum by(uri,method)(rate(http_server_requests_seconds_count{service="$SERVICE",status=~"5.."}[5m]))
)
```

### Phase 6: Dependency Health Check

```sql
-- Get service dependencies
SELECT dependency_type, dependency_name, endpoint,
       timeout_ms, circuit_breaker_enabled
FROM service_dependencies
WHERE service_name = '$SERVICE';
```

**Database Client Metrics**:
```promql
hikaricp_connections_active{service="$SERVICE"}
hikaricp_connections_pending{service="$SERVICE"}
hikaricp_connections_timeout_total{service="$SERVICE"}
```

**Redis Client Metrics**:
```promql
lettuce_command_completion_seconds{service="$SERVICE"}
lettuce_command_firstresponse_seconds{service="$SERVICE"}
```

**HTTP Client Metrics**:
```promql
http_client_requests_seconds{service="$SERVICE"}
```

### Phase 7: CloudWatch Log Analysis

**Search for Exceptions**:
```
fields @timestamp, @message
| filter @message like /Exception|Error|FATAL/
| filter service = "$SERVICE"
| sort @timestamp desc
| limit 50
```

**Search for Specific Patterns**:
- `OutOfMemoryError` - JVM memory exhaustion
- `Connection refused` - Dependency failures
- `TimeoutException` - Upstream timeouts
- `CircuitBreakerOpenException` - Circuit breaker triggered

### Phase 8: Historical Alert Correlation

```sql
-- Find related alerts from same timeframe
SELECT alert_name, alert_value, fired_at, resolved_at
FROM alerts
WHERE service_name = '$SERVICE'
  AND fired_at > NOW() - INTERVAL 24 HOUR
ORDER BY fired_at DESC;

-- Check for recurring patterns
SELECT alert_name, COUNT(*) as occurrences,
       AVG(TIMESTAMPDIFF(MINUTE, fired_at, COALESCE(resolved_at, NOW()))) as avg_duration_min
FROM alerts
WHERE service_name = '$SERVICE'
  AND fired_at > NOW() - INTERVAL 7 DAY
GROUP BY alert_name
ORDER BY occurrences DESC;
```

## Root Cause Patterns

| Symptom Pattern | Likely Cause | Action |
|-----------------|--------------|--------|
| High heap + frequent GC | Memory leak or undersized heap | Heap dump analysis, increase heap |
| High CPU + low throughput | Thread contention or infinite loop | Thread dump analysis |
| Latency spike + normal CPU | External dependency slow | Check DB/cache/API latency |
| Error spike + connection errors | Dependency failure | Check downstream services |
| GC pause > 1s | Full GC triggered | Tune GC or increase heap |
| Thread pool exhausted | Blocking operations | Add async processing |

## Cross-Skill Integration

If investigation reveals:
- **Database latency issues** → Suggest `/investigate-rds`
- **Cache issues** → Suggest `/investigate-redis`
- **Pod resource issues** → Suggest `/investigate-k8s`
- **Host resource issues** → Suggest `/investigate-ec2`
- **Search latency** → Suggest `/investigate-elasticsearch`

## Error Handling

| Failure Mode | Fallback Action |
|--------------|----------------|
| Prometheus unavailable | Check CloudWatch custom metrics |
| iZeus APM gaps | Use standard JVM metrics |
| Logs inaccessible | Check pod logs via K8s API |

## Escalation Criteria

Escalate immediately if:
- L0 service error rate > 5%
- All instances affected simultaneously
- OOM on multiple instances
- Circuit breakers open on critical paths

## Report Template

```markdown
# APM Investigation Report

## Alert Summary
- **Service**: [Service Name]
- **Instance(s)**: [List or count]
- **Priority**: [L0/L1/L2]
- **Alert Type**: [Category]
- **Duration**: [Time since trigger]

## JVM Status
| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| Heap Used | X% | 90% | [OK/WARN/CRIT] |
| CPU | X% | 80% | [OK/WARN/CRIT] |
| GC Pause (avg) | Xms | 200ms | [OK/WARN/CRIT] |
| Live Threads | X | Y max | [OK/WARN] |

## Performance Metrics
| Metric | Current | SLA | Status |
|--------|---------|-----|--------|
| Latency p99 | Xms | Yms | [OK/WARN/CRIT] |
| Error Rate | X% | 1% | [OK/WARN/CRIT] |
| Throughput | X req/s | - | - |

## Top Error Endpoints
| Endpoint | Method | Error Rate | Sample Error |
|----------|--------|------------|--------------|
| /api/x | POST | X% | [error type] |

## Dependency Status
| Dependency | Type | Latency | Status |
|------------|------|---------|--------|
| [name] | DB/Cache/API | Xms | [OK/WARN] |

## Log Analysis
[Key errors and stack traces found]

## Root Cause
[Determined cause with evidence]

## Recommendations
1. **Immediate**: [Restart / Scale / Rollback]
2. **Short-term**: [Code fix / Config change]
3. **Long-term**: [Architecture improvement]
```
