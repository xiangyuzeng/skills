# Redis Alert Investigation Skill

> Version: 1.0 | Edition: Redis/ElastiCache Diagnosis | Platform: AWS ElastiCache / Redis Clusters

## Skill Trigger

Use this skill when you receive alerts related to Redis clusters, including:
- **CPU Alerts**: `Redis CPU使用率大于90%`, `Redis CPU使用率持续3分钟超过70%`
- **Memory Alerts**: `Redis 内存使用率持续3分钟超过70%`, `key淘汰`
- **Connection Alerts**: `Redis 实例连接数使用率大于30%`, `客户端堵塞`
- **Performance Alerts**: `Redis 实例命令平均时延大于2ms`, `客户端缓冲超过32m`
- **Network Alerts**: `Redis 实例流量大于32Mbps`
- **Availability Alerts**: `Redis 实例采集失败，请检查是否存活`

---

## Verified Data Sources

**Prometheus Redis Datasource**:
- `prometheus_redis` (UID: `ff6p0gjt24phce`) - PRIMARY for Redis metrics

**Redis Clusters** (74 available via mcp-db-gateway):

| Category | Clusters |
|----------|----------|
| **API/Gateway** | `luckyus-apigateway`, `luckyus-auth`, `luckyus-authservice`, `luckyus-unionauth` |
| **Sales** | `luckyus-isales-crm`, `luckyus-isales-order`, `luckyus-isales-market`, `luckyus-isales-member` |
| **SCM** | `luckyus-scm-asset`, `luckyus-scm-commodity`, `luckyus-scm-ordering`, `luckyus-scm-shopstock` |
| **Platform** | `luckyus-devops`, `luckyus-cmdb`, `luckyus-koala`, `luckyus-ldas` |
| **Finance** | `luckyus-ifiaccounting`, `luckyus-ifichargecontrol`, `luckyus-ifitax` |
| **AI/Data** | `luckyus-redis-dify`, `luckyus-bigdata-cyberdata`, `luckyus-bigdata-dataplatform` |

**AWS CloudWatch**:
- ElastiCache metrics: `AWS/ElastiCache` namespace
- CloudWatch alarms for Redis clusters

**MySQL DevOps Database**:
- Server: `aws-luckyus-devops-rw` (service topology, alert logs)

---

## Investigation Protocol

### Phase 0: Alert Validation

**Objective**: Extract and validate alert metadata

**Actions**:
1. Parse the alert payload to extract:
   - `alertname`: The specific alert type
   - `instance` or `cluster`: Redis cluster identifier
   - `severity`: Alert level (critical, warning, info)
   - `service_name`: Associated service name (if available)
   - `startsAt`: Alert start timestamp

2. Identify Redis cluster from alert:
```
# Common patterns:
# - luckyus-isales-order
# - luckyus-apigateway
# - luckyus-auth
```

3. Determine service priority level:
```sql
-- Query service level from DevOps DB
-- Server: aws-luckyus-devops-rw
SELECT service_name, level, owner, oncall_group
FROM service_registry
WHERE service_name LIKE '%<cluster_name>%';
```

**Priority Classification**:
| Priority | Response Time | Examples |
|----------|---------------|----------|
| L0 | < 15 minutes | isales-order, auth, apigateway |
| L1 | < 30 minutes | scm-shopstock, devops, session |
| L2 | < 2 hours | dify, bigdata, koala |

---

### Phase 1: Data Availability Check

**Objective**: Verify Redis metrics are accessible

**Actions**:

1. **Check Prometheus Redis Datasource Connectivity**:
```promql
-- Use Grafana MCP: mcp__grafana__query_prometheus
-- Datasource UID: ff6p0gjt24phce (prometheus_redis)
redis_up{instance=~".*<cluster_name>.*"}
```

2. **Verify Redis Exporter Status**:
```promql
redis_exporter_build_info{instance=~".*<cluster_name>.*"}
```

3. **List available Redis instances for the cluster**:
```promql
redis_instance_info{instance=~".*<cluster_name>.*"}
```

**If data unavailable**: Document gap and use direct Redis commands or CloudWatch as fallback.

---

### Phase 2: Redis Health Assessment

**Objective**: Collect comprehensive Redis metrics

**Execute ALL queries in parallel using mcp__grafana__query_prometheus**:

#### 2a. Memory Metrics
```promql
# Memory Usage Percentage (most critical)
redis_memory_used_bytes{instance=~".*<cluster>.*"} / redis_memory_max_bytes{instance=~".*<cluster>.*"} * 100

# Memory Components
redis_memory_used_bytes{instance=~".*<cluster>.*"}
redis_memory_max_bytes{instance=~".*<cluster>.*"}
redis_memory_used_rss_bytes{instance=~".*<cluster>.*"}

# Memory Fragmentation Ratio (>1.5 indicates fragmentation issues)
redis_allocator_frag_ratio{instance=~".*<cluster>.*"}

# Evicted Keys (indicates memory pressure)
rate(redis_evicted_keys_total{instance=~".*<cluster>.*"}[5m])
```

#### 2b. CPU Metrics
```promql
# CPU Usage Rate
rate(redis_cpu_sys_seconds_total{instance=~".*<cluster>.*"}[5m])
rate(redis_cpu_user_seconds_total{instance=~".*<cluster>.*"}[5m])

# Total CPU (sys + user)
rate(redis_cpu_sys_seconds_total{instance=~".*<cluster>.*"}[5m]) + rate(redis_cpu_user_seconds_total{instance=~".*<cluster>.*"}[5m])
```

#### 2c. Connection Metrics
```promql
# Connected Clients
redis_connected_clients{instance=~".*<cluster>.*"}

# Blocked Clients (clients waiting on BLPOP, BRPOP, etc.)
redis_blocked_clients{instance=~".*<cluster>.*"}

# Connection Rate
rate(redis_connections_received_total{instance=~".*<cluster>.*"}[5m])

# Client Buffer Usage
redis_client_recent_max_output_buffer_bytes{instance=~".*<cluster>.*"}
redis_client_recent_max_input_buffer_bytes{instance=~".*<cluster>.*"}
```

#### 2d. Performance Metrics
```promql
# Commands Processed Rate
rate(redis_commands_processed_total{instance=~".*<cluster>.*"}[5m])

# Command Latency (average)
rate(redis_commands_duration_seconds_total{instance=~".*<cluster>.*"}[5m]) / rate(redis_commands_total{instance=~".*<cluster>.*"}[5m])

# Slow Commands (rejected calls may indicate issues)
rate(redis_commands_rejected_calls_total{instance=~".*<cluster>.*"}[5m])
rate(redis_commands_failed_calls_total{instance=~".*<cluster>.*"}[5m])

# Key Space Operations
redis_db_keys{instance=~".*<cluster>.*"}
redis_db_keys_expiring{instance=~".*<cluster>.*"}
```

#### 2e. Network Metrics
```promql
# Network I/O
rate(redis_net_input_bytes_total{instance=~".*<cluster>.*"}[5m])
rate(redis_net_output_bytes_total{instance=~".*<cluster>.*"}[5m])
```

#### 2f. Replication Metrics (if replica)
```promql
# Replication Status
redis_connected_slaves{instance=~".*<cluster>.*"}
redis_connected_slave_lag_seconds{instance=~".*<cluster>.*"}
redis_connected_slave_offset_bytes{instance=~".*<cluster>.*"}
```

---

### Phase 3: Direct Redis Diagnostics

**Objective**: Execute direct Redis commands for real-time status

**Use mcp__mcp-db-gateway__redis_command**:

#### 3a. Memory Info
```
-- Server: <cluster_name> (e.g., luckyus-isales-order)
INFO memory
```

Key fields to extract:
- `used_memory`: Current memory usage
- `used_memory_peak`: Peak memory usage
- `maxmemory`: Configured max memory
- `mem_fragmentation_ratio`: Memory fragmentation
- `evicted_keys`: Number of evicted keys

#### 3b. Client Info
```
INFO clients
```

Key fields:
- `connected_clients`: Current connections
- `blocked_clients`: Blocked clients
- `client_recent_max_output_buffer`: Max output buffer

#### 3c. Stats Info
```
INFO stats
```

Key fields:
- `total_commands_processed`: Total commands
- `instantaneous_ops_per_sec`: Current OPS
- `rejected_connections`: Rejected connections
- `evicted_keys`: Evicted keys count
- `keyspace_hits/misses`: Cache hit rate

#### 3d. Slow Log
```
SLOWLOG GET 10
```

Analyze slow commands for:
- Command type (KEYS, SMEMBERS, LRANGE, etc.)
- Execution time
- Arguments

#### 3e. Client List (for blocked clients investigation)
```
CLIENT LIST
```

Check for:
- Long-idle connections
- High buffer usage clients
- Blocked clients

---

### Phase 4: Service Dependency Analysis

**Objective**: Identify services using this Redis cluster and check their health

**Actions**:

1. **Query service-to-cache mapping**:
```sql
-- Server: aws-luckyus-devops-rw
SELECT service_name, redis_cluster, connection_pool_size, purpose
FROM service_cache_mapping
WHERE redis_cluster LIKE '%<cluster_name>%';
```

2. **Common Redis-to-Service Mappings**:

| Redis Cluster | Primary Services | Purpose |
|---------------|------------------|---------|
| `luckyus-apigateway` | API Gateway | Rate limiting, session |
| `luckyus-auth` | Authentication | Token cache, session |
| `luckyus-isales-order` | Order Service | Order cache, inventory |
| `luckyus-isales-crm` | CRM Service | Customer data cache |
| `luckyus-session` | All web services | Session storage |

3. **Check dependent service health** (if service identified):
```promql
-- Datasource UID: df8o21agxtkw0d (UMBQuerier-Luckin)
up{job=~".*<service_name>.*"}
```

---

### Phase 5: AWS CloudWatch Integration

**Objective**: Cross-reference with AWS ElastiCache metrics

**Actions**:

1. **Get CloudWatch ElastiCache Metrics** (use mcp__cloudwatch-server__get_metric_data):
```
Namespace: AWS/ElastiCache
Metrics:
  - CPUUtilization
  - DatabaseMemoryUsagePercentage
  - CurrConnections
  - NetworkBytesIn
  - NetworkBytesOut
  - CacheHitRate
  - Evictions
  - ReplicationLag
Dimensions: CacheClusterId=<cluster_id>
Period: 300 seconds
```

2. **Check Prometheus AWS ElastiCache metrics**:
```promql
-- Datasource UID: df8o21agxtkw0d
aws_elasticache_cpuutilization_average{cache_cluster_id=~".*<cluster>.*"}
aws_elasticache_database_memory_usage_percentage_average{cache_cluster_id=~".*<cluster>.*"}
aws_elasticache_engine_cpuutilization_average{cache_cluster_id=~".*<cluster>.*"}
```

3. **Check for recent alarms**:
Use mcp__cloudwatch-server__get_active_alarms to find related alarms.

---

### Phase 6: Alert History Analysis

**Objective**: Analyze historical alert patterns

**Actions**:

1. **Query alert history from DevOps DB**:
```sql
-- Server: aws-luckyus-devops-rw
SELECT alert_name, alert_status, instance, severity, create_time
FROM t_umb_alert_log
WHERE instance LIKE '%<cluster_name>%'
ORDER BY create_time DESC
LIMIT 50;
```

2. **Identify recurring patterns**:
```sql
SELECT DATE(create_time) as alert_date, alert_name, COUNT(*) as count
FROM t_umb_alert_log
WHERE instance LIKE '%<cluster_name>%'
  AND create_time > DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY DATE(create_time), alert_name
ORDER BY alert_date DESC, count DESC;
```

---

### Phase 7: Output Report Generation

**Objective**: Generate structured investigation report

**Report Template**:

```markdown
## Redis Investigation Report

### Alert Summary
| Field | Value |
|-------|-------|
| Alert Name | [alertname] |
| Redis Cluster | [cluster_name] |
| Instance | [instance_ip] |
| Severity | [L0/L1/L2] |
| Start Time | [timestamp] |
| Duration | [duration] |

### Service Context
- **Service Level**: [L0/L1/L2]
- **Dependent Services**: [service_list]
- **Service Owner**: [owner]
- **On-Call Group**: [oncall_group]

### Investigation Findings

#### Phase 1: Data Availability
- Prometheus Redis: [Available/Unavailable]
- Redis Exporter: [Running/Down]
- Direct Redis Access: [Available/Unavailable]

#### Phase 2: Redis Health Metrics
| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| Memory Usage | [X%] | 70% | [OK/WARNING/CRITICAL] |
| CPU Usage | [X%] | 70% | [OK/WARNING/CRITICAL] |
| Connected Clients | [X] | [max_clients] | [OK/WARNING/CRITICAL] |
| Blocked Clients | [X] | 0 | [OK/WARNING/CRITICAL] |
| Command Latency | [X ms] | 2ms | [OK/WARNING/CRITICAL] |
| Eviction Rate | [X/s] | 0 | [OK/WARNING/CRITICAL] |
| Memory Fragmentation | [X] | 1.5 | [OK/WARNING/CRITICAL] |
| Network In | [X MB/s] | 32MB/s | [OK/WARNING/CRITICAL] |
| Network Out | [X MB/s] | 32MB/s | [OK/WARNING/CRITICAL] |

#### Phase 3: Direct Redis Diagnostics
- **Slow Commands Found**: [Yes/No]
- **Top Slow Commands**: [list]
- **Hot Keys Identified**: [Yes/No]
- **Client Buffer Issues**: [Yes/No]

#### Phase 4: Dependent Services
| Service | Status | Connection Pool | Health |
|---------|--------|-----------------|--------|
| [service1] | [Active] | [X/Y] | [Healthy/Degraded] |
| [service2] | [Active] | [X/Y] | [Healthy/Degraded] |

#### Phase 5: CloudWatch Correlation
- [CloudWatch findings and correlations]

#### Phase 6: Alert History
- **Recent Alert Count**: [X in last 24h]
- **Pattern Detected**: [Yes/No - describe pattern]

### Root Cause Analysis
**Primary Cause**: [Identified root cause]
**Contributing Factors**: [Additional factors]

### Recommendations
| Priority | Action | Owner | Timeline |
|----------|--------|-------|----------|
| P0 | [Immediate action] | [team] | Immediate |
| P1 | [Short-term fix] | [team] | 24 hours |
| P2 | [Long-term solution] | [team] | 1 week |

### Evidence
<details>
<summary>Prometheus Queries and Results</summary>

[Include actual query results]

</details>

<details>
<summary>Direct Redis Commands Output</summary>

[Include INFO, SLOWLOG results]

</details>

<details>
<summary>CloudWatch Data</summary>

[Include CloudWatch metrics]

</details>

---
*Investigation completed at [timestamp]*
*Skill Version: Redis Alert Investigation SOP v1.0*
```

---

## MCP Tools Reference

| Tool | Server | Purpose |
|------|--------|---------|
| `mcp__grafana__query_prometheus` | grafana | Redis metrics queries |
| `mcp__grafana__list_prometheus_label_values` | grafana | Discover Redis clusters |
| `mcp__mcp-db-gateway__redis_command` | mcp-db-gateway | Direct Redis diagnostics |
| `mcp__mcp-db-gateway__mysql_query` | mcp-db-gateway | Service topology lookup |
| `mcp__cloudwatch-server__get_metric_data` | cloudwatch-server | AWS ElastiCache metrics |
| `mcp__cloudwatch-server__get_active_alarms` | cloudwatch-server | Related alarms |

## Key Datasource UIDs

| Datasource | UID | Use |
|------------|-----|-----|
| prometheus_redis | `ff6p0gjt24phce` | Primary Redis metrics |
| UMBQuerier-Luckin | `df8o21agxtkw0d` | AWS ElastiCache metrics |

## DevOps Database

| Server | Purpose |
|--------|---------|
| `aws-luckyus-devops-rw` | Service registry, alert logs, cache mappings |

---

## Common Redis Alert Scenarios

### Scenario 1: Memory Usage High
**Symptoms**: Memory > 70%, evictions occurring
**Investigation Focus**:
- Check memory fragmentation ratio
- Identify large keys: `DEBUG OBJECT <key>`
- Check TTL distribution
- Review eviction policy

### Scenario 2: High Latency
**Symptoms**: Command latency > 2ms
**Investigation Focus**:
- Check SLOWLOG for problematic commands
- Look for KEYS, SMEMBERS on large sets
- Check network issues
- Review client output buffer

### Scenario 3: Connection Exhaustion
**Symptoms**: Connected clients approaching limit
**Investigation Focus**:
- CLIENT LIST to identify connection sources
- Check for connection leaks in applications
- Review connection pool settings
- Check for blocked clients

### Scenario 4: Client Blocking
**Symptoms**: Blocked clients > 0
**Investigation Focus**:
- Identify blocking operations (BLPOP, BRPOP)
- Check if producers are running
- Review application logic

---

## Example Usage

When investigating a Redis alert, invoke this skill with the alert payload:

```
/investigate-redis {
  "alertname": "Redis 内存使用率持续3分钟超过70%",
  "instance": "luckyus-isales-order",
  "severity": "warning",
  "current_value": 75
}
```

The skill will execute all 7 phases automatically and generate a comprehensive report.
