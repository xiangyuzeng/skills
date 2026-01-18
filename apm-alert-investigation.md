# Application/APM Alert Investigation Skill

> Version: 1.0 | Edition: Application Performance Monitoring | Platform: iZeus APM / Spring Boot Services

## Skill Trigger

Use this skill when you receive alerts related to application performance and JVM metrics, including:
- **JVM Alerts**: `JVM CPU使用率大于20%`, `FGC 次数大于0`, `YGC 耗时大于500毫秒`
- **Response Time Alerts**: `服务响应时间（ms）大于1500`
- **Error Alerts**: `服务每分钟异常数大于X`, `端点每分钟失败数大于X`
- **HTTP Alerts**: `异常okhttp总数大于等于50`
- **Gateway Alerts**: `网关告警 错误率大于15%`
- **iZeus Strategy Alerts**: `【iZeus-策略X】` series alerts

---

## Verified Data Sources

**Prometheus Datasources**:
- `UMBQuerier-Luckin` (UID: `df8o21agxtkw0d`) - PRIMARY for JVM and application metrics

**Available JVM Metrics**:
```promql
# Memory metrics
jvm_memory_used_bytes, jvm_memory_max_bytes, jvm_memory_committed_bytes
jvm_memory_pool_bytes_used, jvm_memory_pool_bytes_max

# GC metrics
jvm_gc_collection_seconds_count, jvm_gc_collection_seconds_sum
jvm_gc_pause_seconds_count, jvm_gc_pause_seconds_sum
jvm_gc_live_data_size_bytes, jvm_gc_max_data_size_bytes
jvm_gc_overhead_percent

# Thread metrics
jvm_threads_current, jvm_threads_peak, jvm_threads_daemon
jvm_threads_deadlocked, jvm_threads_live_threads

# Class loading
jvm_classes_loaded, jvm_classes_currently_loaded

# Buffer pools
jvm_buffer_memory_used_bytes, jvm_buffer_pool_used_bytes
```

**MySQL DevOps Database**:
- Server: `aws-luckyus-devops-rw` (service topology, alert logs)

**AWS CloudWatch**:
- Application logs for Spring Boot services

---

## Investigation Protocol

### Phase 0: Alert Validation

**Objective**: Extract and validate alert metadata

**Actions**:
1. Parse the alert payload to extract:
   - `alertname`: The specific alert type (iZeus strategy, JVM, response time, etc.)
   - `service_name` or `application`: Service identifier
   - `instance`: Instance IP:port
   - `endpoint`: Specific API endpoint (if applicable)
   - `severity`: Alert level
   - `startsAt`: Alert start timestamp

2. Identify alert category:

| Alert Pattern | Category | Priority |
|---------------|----------|----------|
| `iZeus-策略X` | iZeus Strategy Alert | Based on strategy level |
| `FGC`, `YGC`, `GC` | Garbage Collection | L0/L1 |
| `JVM CPU`, `JVM Memory` | JVM Resource | L1 |
| `响应时间`, `Response Time` | Latency | L0/L1 |
| `异常数`, `Exception` | Error Rate | L0/L1 |
| `失败数`, `Failure` | Endpoint Failure | L0/L1 |
| `网关`, `Gateway` | API Gateway | L0 |

3. Determine service priority level:
```sql
-- Query service level from DevOps DB
-- Server: aws-luckyus-devops-rw
SELECT service_name, level, owner, oncall_group
FROM service_registry
WHERE service_name = '<service_name>';
```

**Priority Classification**:
| Priority | Response Time | Examples |
|----------|---------------|----------|
| L0 | < 15 minutes | isalesorderservice, isalespaymentservice, apigateway |
| L1 | < 30 minutes | isalescrm, iscmcommodity, notification services |
| L2 | < 2 hours | Background services, analytics, admin tools |

---

### Phase 1: Data Availability Check

**Objective**: Verify APM metrics are accessible

**Actions**:

1. **Check Service Up Status**:
```promql
-- Use Grafana MCP: mcp__grafana__query_prometheus
-- Datasource UID: df8o21agxtkw0d (UMBQuerier-Luckin)
up{job=~".*<service_name>.*"}
```

2. **Verify JVM Metrics Available**:
```promql
jvm_info{application=~".*<service_name>.*"}
```

3. **Check iZeus/SkyWalking Data**:
```promql
# Check if APM data is flowing
service_instance_sla{service=~".*<service_name>.*"}
```

**If data unavailable**: Document gap and use CloudWatch logs as fallback.

---

### Phase 2: JVM Health Assessment

**Objective**: Collect comprehensive JVM metrics

**Execute ALL queries in parallel using mcp__grafana__query_prometheus**:

#### 2a. Memory Metrics
```promql
# Heap Memory Usage
jvm_memory_used_bytes{application=~".*<service>.*",area="heap"} / jvm_memory_max_bytes{application=~".*<service>.*",area="heap"} * 100

# Memory by Pool
jvm_memory_pool_bytes_used{application=~".*<service>.*"}
jvm_memory_pool_bytes_max{application=~".*<service>.*"}

# Non-Heap Memory (Metaspace, Code Cache)
jvm_memory_used_bytes{application=~".*<service>.*",area="nonheap"}

# Memory after GC (indicates memory leaks if growing)
jvm_memory_usage_after_gc_percent{application=~".*<service>.*"}
```

#### 2b. Garbage Collection Metrics
```promql
# GC Count Rate (FGC and YGC)
rate(jvm_gc_collection_seconds_count{application=~".*<service>.*"}[5m])

# GC Duration Rate (total time spent in GC)
rate(jvm_gc_collection_seconds_sum{application=~".*<service>.*"}[5m])

# GC Overhead (percentage of time in GC)
jvm_gc_overhead_percent{application=~".*<service>.*"}

# GC Pause Time
jvm_gc_pause_seconds_sum{application=~".*<service>.*"}
jvm_gc_pause_seconds_count{application=~".*<service>.*"}

# Live Data Size (long-lived objects)
jvm_gc_live_data_size_bytes{application=~".*<service>.*"}
jvm_gc_max_data_size_bytes{application=~".*<service>.*"}
```

#### 2c. Thread Metrics
```promql
# Current Thread Count
jvm_threads_current{application=~".*<service>.*"}
jvm_threads_live_threads{application=~".*<service>.*"}

# Peak Threads
jvm_threads_peak{application=~".*<service>.*"}

# Daemon Threads
jvm_threads_daemon{application=~".*<service>.*"}

# Deadlocked Threads (CRITICAL if > 0)
jvm_threads_deadlocked{application=~".*<service>.*"}
```

#### 2d. CPU Metrics
```promql
# Process CPU Usage
process_cpu_usage{application=~".*<service>.*"}

# System CPU Usage
system_cpu_usage{application=~".*<service>.*"}

# Load Average
system_load_average_1m{application=~".*<service>.*"}
```

#### 2e. Class Loading
```promql
# Loaded Classes
jvm_classes_currently_loaded{application=~".*<service>.*"}

# Total Classes Loaded
jvm_classes_loaded_total{application=~".*<service>.*"}

# Unloaded Classes
jvm_classes_unloaded_total{application=~".*<service>.*"}
```

---

### Phase 3: Application Performance Analysis

**Objective**: Analyze application-level performance metrics

#### 3a. Response Time Analysis
```promql
# HTTP Request Duration (if using Spring Boot Actuator)
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{application=~".*<service>.*"}[5m]))
histogram_quantile(0.99, rate(http_server_requests_seconds_bucket{application=~".*<service>.*"}[5m]))

# Average Response Time
rate(http_server_requests_seconds_sum{application=~".*<service>.*"}[5m]) / rate(http_server_requests_seconds_count{application=~".*<service>.*"}[5m])
```

#### 3b. Error Rate Analysis
```promql
# HTTP Error Rate (5xx errors)
sum(rate(http_server_requests_seconds_count{application=~".*<service>.*",status=~"5.."}[5m])) / sum(rate(http_server_requests_seconds_count{application=~".*<service>.*"}[5m])) * 100

# Client Error Rate (4xx errors)
sum(rate(http_server_requests_seconds_count{application=~".*<service>.*",status=~"4.."}[5m])) / sum(rate(http_server_requests_seconds_count{application=~".*<service>.*"}[5m])) * 100

# Total Request Rate
sum(rate(http_server_requests_seconds_count{application=~".*<service>.*"}[5m]))
```

#### 3c. Endpoint-Specific Analysis
```promql
# Slowest Endpoints
topk(10, histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{application=~".*<service>.*"}[5m])))

# Highest Error Rate Endpoints
topk(10, sum by (uri) (rate(http_server_requests_seconds_count{application=~".*<service>.*",status=~"5.."}[5m])))
```

#### 3d. Connection Pool Metrics
```promql
# Database Connection Pool (HikariCP)
hikaricp_connections_active{application=~".*<service>.*"}
hikaricp_connections_idle{application=~".*<service>.*"}
hikaricp_connections_pending{application=~".*<service>.*"}
hikaricp_connections_max{application=~".*<service>.*"}

# Connection Wait Time
hikaricp_connections_acquire_seconds_sum{application=~".*<service>.*"}
```

---

### Phase 4: iZeus APM Deep Dive

**Objective**: Analyze iZeus/SkyWalking specific metrics

#### 4a. Service Level Metrics
```promql
# Service SLA (Success Rate)
service_sla{service=~".*<service>.*"}

# Service Calls Per Minute (CPM)
service_cpm{service=~".*<service>.*"}

# Service Response Time (avg, P50, P75, P90, P95, P99)
service_resp_time{service=~".*<service>.*"}
service_percentile{service=~".*<service>.*"}
```

#### 4b. Endpoint Level Metrics
```promql
# Endpoint SLA
endpoint_sla{service=~".*<service>.*",endpoint=~".*<endpoint>.*"}

# Endpoint CPM
endpoint_cpm{service=~".*<service>.*",endpoint=~".*<endpoint>.*"}

# Endpoint Response Time
endpoint_resp_time{service=~".*<service>.*",endpoint=~".*<endpoint>.*"}
```

#### 4c. Instance Level Metrics
```promql
# Instance JVM CPU
instance_jvm_cpu{service=~".*<service>.*"}

# Instance JVM Memory
instance_jvm_memory_heap_used{service=~".*<service>.*"}
instance_jvm_memory_heap_max{service=~".*<service>.*"}

# Instance JVM GC
instance_jvm_young_gc_count{service=~".*<service>.*"}
instance_jvm_old_gc_count{service=~".*<service>.*"}
```

---

### Phase 5: Dependency Analysis

**Objective**: Analyze downstream dependencies and their health

#### 5a. Database Dependencies
```promql
# Database Call Success Rate
database_access_sla{service=~".*<service>.*"}

# Database Response Time
database_access_resp_time{service=~".*<service>.*"}

# Database Call Rate
database_access_cpm{service=~".*<service>.*"}
```

#### 5b. Cache Dependencies (Redis)
```promql
# Redis Call Metrics
redis_access_sla{service=~".*<service>.*"}
redis_access_resp_time{service=~".*<service>.*"}
```

#### 5c. Service-to-Service Dependencies
```promql
# Downstream Service Calls
service_relation_server_call_sla{service=~".*<service>.*"}
service_relation_server_resp_time{service=~".*<service>.*"}

# Upstream Service Calls
service_relation_client_call_sla{dest_service=~".*<service>.*"}
```

---

### Phase 6: Root Cause Analysis by Alert Type

**Objective**: Targeted investigation based on alert category

#### 6a. JVM GC Issues (FGC/YGC)

**Check for**:
```promql
# High GC overhead
jvm_gc_overhead_percent{application=~".*<service>.*"} > 10

# Memory after GC growing (memory leak indicator)
increase(jvm_gc_live_data_size_bytes{application=~".*<service>.*"}[1h])

# Frequent Full GC
rate(jvm_gc_collection_seconds_count{application=~".*<service>.*",gc="G1 Old Generation"}[5m])
```

**Common Causes**:
| Symptom | Likely Cause | Action |
|---------|--------------|--------|
| High YGC frequency | Eden space too small | Increase heap or tune `-XX:NewRatio` |
| High FGC frequency | Memory leak or heap too small | Heap dump analysis, increase heap |
| Long GC pauses | Large heap with stop-the-world GC | Switch to G1/ZGC, tune pause targets |
| Memory after GC growing | Memory leak | Heap dump, profiling |

#### 6b. High Response Time

**Check for**:
```promql
# Identify slow endpoints
topk(5, histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{application=~".*<service>.*"}[5m])))

# Check database latency
database_access_resp_time{service=~".*<service>.*"}

# Check downstream service latency
service_relation_server_resp_time{service=~".*<service>.*"}
```

**Common Causes**:
| Symptom | Likely Cause | Action |
|---------|--------------|--------|
| All endpoints slow | GC pause, CPU saturation | Check JVM metrics |
| Specific endpoint slow | Slow query, external call | Analyze traces |
| Intermittent slowness | Connection pool exhaustion | Check connection pools |

#### 6c. High Error Rate

**Check for**:
```promql
# Error rate by status code
sum by (status) (rate(http_server_requests_seconds_count{application=~".*<service>.*",status=~"[45].."}[5m]))

# Error rate by endpoint
sum by (uri) (rate(http_server_requests_seconds_count{application=~".*<service>.*",status=~"5.."}[5m]))
```

**Common Causes**:
| Error Type | Likely Cause | Action |
|------------|--------------|--------|
| 500 errors | Application bug, dependency failure | Check logs, traces |
| 503 errors | Service overloaded | Scale up, circuit breaker |
| Connection errors | Pool exhausted, network issue | Check pools, network |

---

### Phase 7: CloudWatch Log Analysis

**Objective**: Search application logs for error details

**Actions**:

1. **Query application logs** (use mcp__cloudwatch-server__execute_log_insights_query):
```
-- Find exceptions and errors
fields @timestamp, @message
| filter @message like /(?i)(exception|error|failed|timeout)/
| sort @timestamp desc
| limit 100
```

2. **Find specific error patterns**:
```
-- OutOfMemoryError
fields @timestamp, @message
| filter @message like /OutOfMemoryError/
| sort @timestamp desc
| limit 20

-- Connection pool exhaustion
fields @timestamp, @message
| filter @message like /(?i)(connection.*pool|pool.*exhausted|timeout.*connection)/
| sort @timestamp desc
| limit 20
```

3. **Correlate with alert time**:
```
-- Get logs around alert time
fields @timestamp, @message
| filter @timestamp >= '<alert_start - 5min>' and @timestamp <= '<alert_start + 5min>'
| filter @message like /(?i)(error|exception|warn)/
| sort @timestamp asc
| limit 200
```

---

### Phase 8: Alert History Analysis

**Objective**: Analyze historical alert patterns

**Actions**:

1. **Query alert history from DevOps DB**:
```sql
-- Server: aws-luckyus-devops-rw
SELECT alert_name, alert_status, instance, severity, create_time
FROM t_umb_alert_log
WHERE instance LIKE '%<service_name>%'
   OR alert_name LIKE '%<service_name>%'
ORDER BY create_time DESC
LIMIT 50;
```

2. **Identify recurring patterns**:
```sql
SELECT DATE(create_time) as alert_date, alert_name, COUNT(*) as count
FROM t_umb_alert_log
WHERE (instance LIKE '%<service_name>%' OR alert_name LIKE '%<service_name>%')
  AND create_time > DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY DATE(create_time), alert_name
ORDER BY alert_date DESC, count DESC;
```

---

### Phase 9: Output Report Generation

**Objective**: Generate structured investigation report

**Report Template**:

```markdown
## Application/APM Investigation Report

### Alert Summary
| Field | Value |
|-------|-------|
| Alert Name | [alertname] |
| Service | [service_name] |
| Instance | [instance_ip] |
| Endpoint | [endpoint if applicable] |
| Severity | [L0/L1/L2] |
| Start Time | [timestamp] |
| Duration | [duration] |

### Service Context
- **Service Level**: [L0/L1/L2]
- **Service Owner**: [owner]
- **On-Call Group**: [oncall_group]
- **Dependencies**: [db, cache, downstream services]

### Investigation Findings

#### Phase 1: Data Availability
- Prometheus Metrics: [Available/Unavailable]
- iZeus/APM Data: [Available/Unavailable]
- CloudWatch Logs: [Available/Unavailable]

#### Phase 2: JVM Health
| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| Heap Usage | [X%] | 85% | [OK/WARNING/CRITICAL] |
| GC Overhead | [X%] | 10% | [OK/WARNING/CRITICAL] |
| FGC Rate | [X/min] | 0.1/min | [OK/WARNING/CRITICAL] |
| YGC Duration | [X ms] | 500ms | [OK/WARNING/CRITICAL] |
| Thread Count | [X] | [max] | [OK/WARNING/CRITICAL] |
| Deadlocked Threads | [X] | 0 | [OK/WARNING/CRITICAL] |
| CPU Usage | [X%] | 80% | [OK/WARNING/CRITICAL] |

#### Phase 3: Application Performance
| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| P95 Response Time | [X ms] | 1500ms | [OK/WARNING/CRITICAL] |
| Error Rate (5xx) | [X%] | 1% | [OK/WARNING/CRITICAL] |
| Request Rate | [X/s] | - | [Info] |
| DB Connection Pool | [X/Y] | [max] | [OK/WARNING/CRITICAL] |

#### Phase 4: iZeus APM Metrics
| Metric | Current | Status |
|--------|---------|--------|
| Service SLA | [X%] | [OK/WARNING/CRITICAL] |
| Service CPM | [X] | [Info] |
| Avg Response Time | [X ms] | [OK/WARNING/CRITICAL] |

#### Phase 5: Dependencies
| Dependency | Type | SLA | Response Time | Status |
|------------|------|-----|---------------|--------|
| [db_name] | MySQL | [X%] | [X ms] | [Healthy/Degraded] |
| [cache_name] | Redis | [X%] | [X ms] | [Healthy/Degraded] |
| [service_name] | Service | [X%] | [X ms] | [Healthy/Degraded] |

#### Phase 6: Root Cause Analysis
**Alert Category**: [GC/Response Time/Error Rate/etc.]

**Root Cause**: [Identified cause]

**Evidence**:
- [Key finding 1]
- [Key finding 2]

#### Phase 7: Log Findings
- **Exception Types Found**: [list]
- **Error Count**: [X in investigation window]
- **Key Log Messages**: [summary]

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
<summary>JVM Metrics</summary>

[Include JVM query results]

</details>

<details>
<summary>Application Logs</summary>

[Include relevant log entries]

</details>

<details>
<summary>APM Traces</summary>

[Include trace information if available]

</details>

---
*Investigation completed at [timestamp]*
*Skill Version: APM Alert Investigation SOP v1.0*
```

---

## MCP Tools Reference

| Tool | Server | Purpose |
|------|--------|---------|
| `mcp__grafana__query_prometheus` | grafana | JVM and APM metrics |
| `mcp__grafana__list_prometheus_label_values` | grafana | Discover services/instances |
| `mcp__mcp-db-gateway__mysql_query` | mcp-db-gateway | Service topology, alert logs |
| `mcp__cloudwatch-server__execute_log_insights_query` | cloudwatch-server | Application logs |
| `mcp__cloudwatch-server__get_active_alarms` | cloudwatch-server | Related alarms |

## Key Datasource UIDs

| Datasource | UID | Use |
|------------|-----|-----|
| UMBQuerier-Luckin | `df8o21agxtkw0d` | Primary JVM and APM metrics |

## DevOps Database

| Server | Purpose |
|--------|---------|
| `aws-luckyus-devops-rw` | Service registry, alert logs |

---

## Common APM Alert Scenarios

### Scenario 1: Full GC (FGC) Alerts
**Symptoms**: `FGC 次数大于0`, high GC overhead
**Investigation Focus**:
- Check heap usage trend
- Check memory after GC (leak indicator)
- Review recent deployments
- Consider heap dump

### Scenario 2: High Response Time
**Symptoms**: `服务响应时间大于1500ms`
**Investigation Focus**:
- Identify slow endpoints
- Check GC pauses
- Check database latency
- Check downstream dependencies

### Scenario 3: High Exception Rate
**Symptoms**: `服务每分钟异常数大于X`
**Investigation Focus**:
- Check CloudWatch logs for stack traces
- Identify exception types
- Check recent deployments
- Check dependency health

### Scenario 4: Gateway Error Rate
**Symptoms**: `网关告警 错误率大于15%`
**Investigation Focus**:
- Identify failing upstream services
- Check rate limiting
- Check authentication issues
- Check routing configuration

---

## Example Usage

When investigating an APM alert, invoke this skill with the alert payload:

```
/investigate-apm {
  "alertname": "【iZeus-策略1】-服务每分钟异常数大于2",
  "service_name": "isalesorderservice",
  "instance": "10.1.2.3:8080",
  "severity": "warning"
}
```

The skill will execute all 9 phases automatically and generate a comprehensive report.
