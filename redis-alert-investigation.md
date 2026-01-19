---
name: redis-alert-investigation
description: This skill should be used when the user asks to "investigate Redis alert", "debug cache issues", "check Redis cluster", "analyze Redis performance", mentions Redis/ElastiCache issues, cache memory pressure, high latency, connection exhaustion, evictions, or receives alerts about Redis clusters including CPU spikes, memory usage, slow commands, or replication problems.
version: 2.0.0
---

# Redis Alert Investigation Skill v2.0

## Overview

Comprehensive protocol for investigating Redis/ElastiCache alerts with direct Redis diagnostics, Prometheus metrics, and service dependency correlation.

## Quick Reference

| Data Source | Server/UID | Purpose |
|-------------|------------|---------|
| Redis Clusters | 74 available via mcp-db-gateway | Direct commands |
| Prometheus | `ff6p0gjt24phce` | Redis metrics |
| CloudWatch | ElastiCache namespace | AWS-native metrics |
| DevOps DB | `aws-luckyus-devops-rw` | Service topology |

## Alert Categories

| Category | Trigger Conditions | Priority |
|----------|-------------------|----------|
| Memory | > 85% used OR evictions > 100/s | L0/L1 |
| CPU | > 70% sustained | L1/L2 |
| Connections | > 80% max OR blocked clients | L0/L1 |
| Latency | > 2ms average OR slow commands | L1/L2 |
| Replication | Lag > 1s OR link down | L0/L1 |
| Network | Bandwidth > 80% limit | L1 |

## Investigation Protocol

### Phase 1: Alert Validation (Max 30s)

Extract from alert payload:
- `cluster_name`, `node_endpoint`, `node_role`
- Alert type, current value, threshold
- Time since alert triggered

Classify cluster tier:
- **L0**: session-cache, payment-cache, cart-cache
- **L1**: product-cache, user-cache, config-cache
- **L2**: analytics-cache, log-cache

### Phase 2: Parallel Data Collection

**CRITICAL**: Execute these in parallel:

**Prometheus Redis Metrics**:
```promql
# Memory (parallel)
redis_memory_used_bytes{cluster="$CLUSTER"}
redis_memory_max_bytes{cluster="$CLUSTER"}
redis_memory_fragmentation_ratio{cluster="$CLUSTER"}
redis_evicted_keys_total{cluster="$CLUSTER"}

# Performance (parallel)
redis_commands_processed_total{cluster="$CLUSTER"}
redis_connected_clients{cluster="$CLUSTER"}
redis_blocked_clients{cluster="$CLUSTER"}
redis_instantaneous_ops_per_sec{cluster="$CLUSTER"}
```

**Latency Metrics**:
```promql
redis_commands_duration_seconds_total{cluster="$CLUSTER"} / redis_commands_total{cluster="$CLUSTER"}
histogram_quantile(0.99, rate(redis_command_latencies_bucket{cluster="$CLUSTER"}[5m]))
```

### Phase 3: Direct Redis Diagnostics

**INFO Command**:
```
redis_command(server="$CLUSTER", command="INFO", args=["all"])
```

Key sections to analyze:
- `# Memory`: used_memory, maxmemory, mem_fragmentation_ratio
- `# Clients`: connected_clients, blocked_clients
- `# Stats`: instantaneous_ops_per_sec, evicted_keys
- `# Replication`: role, master_link_status, master_last_io_seconds_ago

**SLOWLOG Analysis**:
```
redis_command(server="$CLUSTER", command="SLOWLOG", args=["GET", "20"])
```

Analyze for:
- Commands taking > 10ms
- Frequent slow command patterns (KEYS, SCAN without cursor, large HGETALL)
- Specific keys causing slowness

**CLIENT LIST**:
```
redis_command(server="$CLUSTER", command="CLIENT", args=["LIST"])
```

Check for:
- Connection distribution across services
- Idle connections holding resources
- Blocked clients and blocking commands

### Phase 4: Memory Deep Dive (if memory alert)

**Memory Analysis**:
```
redis_command(server="$CLUSTER", command="MEMORY", args=["STATS"])
redis_command(server="$CLUSTER", command="MEMORY", args=["DOCTOR"])
```

**Big Keys Detection**:
```
redis_command(server="$CLUSTER", command="DEBUG", args=["OBJECT", "FREQ"])
```

Thresholds for concern:
- Fragmentation ratio > 1.5 = external fragmentation issue
- Fragmentation ratio < 1.0 = potential memory swap
- RSS vs used_memory gap > 20% = memory overhead

### Phase 5: Service Dependency Impact

```sql
-- Find services using this Redis cluster
SELECT s.service_name, s.service_priority, s.instance_count,
       srm.cache_purpose
FROM services s
JOIN service_redis_mapping srm ON s.id = srm.service_id
WHERE srm.cluster_name = '$CLUSTER';
```

**Check Dependent Service Health**:
```promql
# Cache hit rate for dependent services
rate(redis_cache_hits_total{service=~"$DEPENDENT_SERVICES"}[5m]) /
(rate(redis_cache_hits_total{service=~"$DEPENDENT_SERVICES"}[5m]) +
 rate(redis_cache_misses_total{service=~"$DEPENDENT_SERVICES"}[5m]))
```

### Phase 6: CloudWatch Correlation

Query ElastiCache metrics:
- `CPUUtilization`
- `EngineCPUUtilization`
- `DatabaseMemoryUsagePercentage`
- `CurrConnections`
- `NetworkBytesIn/Out`
- `ReplicationLag`

Check for AWS events and maintenance windows.

### Phase 7: Replication Analysis (if applicable)

```promql
# Replication metrics
redis_connected_slaves{cluster="$CLUSTER"}
redis_master_repl_offset{cluster="$CLUSTER"} - redis_slave_repl_offset{cluster="$CLUSTER"}
```

Verify:
- Master-replica link status
- Replication backlog size
- Partial sync vs full sync frequency

## Root Cause Patterns

| Symptom Pattern | Likely Cause | Action |
|-----------------|--------------|--------|
| High memory + evictions | Key TTL issues or memory limit | Increase memory or review TTLs |
| High CPU + slow commands | O(N) commands on large keys | Optimize data structures |
| Fragmentation > 1.5 | Key size variations | Consider MEMORY PURGE or restart |
| Blocked clients | BLPOP/BRPOP timeouts | Check producer-consumer balance |
| Connection spikes | Connection pool misconfiguration | Tune pool settings |

## Cross-Skill Integration

If investigation reveals:
- **Database fallback causing load** → Suggest `/investigate-rds`
- **Application timeout handling** → Suggest `/investigate-apm`
- **Host resource issues** → Suggest `/investigate-ec2`

## Error Handling

| Failure Mode | Fallback Action |
|--------------|----------------|
| Redis commands blocked | Use Prometheus metrics only |
| Prometheus gaps | Use CloudWatch ElastiCache metrics |
| Cluster unreachable | Check network/security groups |

## Escalation Criteria

Escalate immediately if:
- L0 cache cluster unreachable
- Eviction rate causing service degradation
- Replication completely broken
- Memory at 100% with eviction policy=noeviction

## Report Template

```markdown
# Redis Investigation Report

## Alert Summary
- **Cluster**: [Cluster Name]
- **Node**: [Primary/Replica] [Endpoint]
- **Tier**: [L0/L1/L2]
- **Alert Type**: [Category]

## Cluster Status
| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| Memory Used | X% | 85% | [OK/WARN/CRIT] |
| CPU | X% | 70% | [OK/WARN/CRIT] |
| Connections | X | Y max | [OK/WARN/CRIT] |
| Latency (p99) | Xms | 2ms | [OK/WARN/CRIT] |

## Slow Commands (Last 10min)
| Command | Duration | Key Pattern |
|---------|----------|-------------|
| [CMD] | [Xms] | [pattern] |

## Memory Analysis
- Used: X MB / Y MB (Z%)
- Fragmentation Ratio: X.XX
- Evictions (last 1h): N

## Root Cause
[Determined cause with evidence]

## Affected Services
| Service | Priority | Cache Purpose | Impact |
|---------|----------|---------------|--------|
| [name] | [L0/L1] | [session/etc] | [description] |

## Recommendations
1. **Immediate**: [Action]
2. **Short-term**: [Action]
3. **Long-term**: [Action]
```
