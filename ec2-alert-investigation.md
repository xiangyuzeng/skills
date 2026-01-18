# EC2 Alert Investigation Skill

> Version: 5.0 | Edition: EC2 Instance Diagnosis | Platform: AWS EC2 Instances

## Skill Trigger

Use this skill when you receive alerts related to EC2 instances, including:

### VM Alert Types (from production alerting)
| Alert Pattern | Chinese Name | Threshold |
|---------------|--------------|-----------|
| `【vm-CPU】P1` | CPU平均负载大于CPU核心数量的1倍 | Load > cores |
| `【vm-CPU】P1` | 服务整体CPU平均使用率超过80% | CPU > 80% |
| `【vm-cpu】P0` | CPU_iowait每秒的使用率大于80% | IOWait > 80% |
| `【vm-cpu】P0` | 服务CPU使用率窃取大于10% | CPU steal > 10% |
| `【vm-内存】P1` | 内存使用率大于90% 持续10分钟 | Memory > 90% |
| `【vm-磁盘】P1` | 分区使用率大于90% | Disk > 90% |
| `【vm-fileSystem】P0` | 分区inodes使用率大于95% | Inodes > 95% |
| `【vm-fileSystem】P0` | 分区发送只读事件 | Read-only FS |
| `【vm-io】P0` | 服务io耗时大于90ms | IO latency > 90ms |
| `【vm-io】P1` | 磁盘IO使用率大于70% | IO util > 70% |
| `【vm-tcp】P0` | TCP每秒重传报文数超过200 | TCP retrans > 200/s |
| `【vm-网卡】P0` | 入/出方向每秒丢弃数据包大于20 | Packet drops > 20/s |
| `【vm-网卡】P0` | 网卡状态为down | NIC down |
| `【vm-宕机】P0` | up监控指标心跳丢失10分钟 | Instance down |
| `【iZeus】Node-*` | Node-CPU-85, Node-Disk-85, Node-Memory-95 | Various |

### Legacy Alert Names
- **Disk Alerts**: `DiskUsedPercent`, `DiskInodesUsedPercent`, disk I/O issues
- **Memory Alerts**: `MemUsedPercent`, OOM events, memory pressure
- **CPU Alerts**: `CPUTotalUsedPercent`, CPU throttling, high load average
- **I/O Alerts**: `DiskIOReadBytes`, `DiskIOWriteBytes`, I/O wait
- **Network Alerts**: `NetBytesIn`, `NetBytesOut`, packet loss, connection issues

---

## Verified Data Sources

**Prometheus** (3 datasources):
- `UMBQuerier-Luckin` (UID: `df8o21agxtkw0d`) - PRIMARY for node metrics
- `prometheus` (UID: `ff7hkeec6c9a8e`) - General metrics
- `prometheus_redis` (UID: `ff6p0gjt24phce`) - Redis metrics (74 clusters)

**Prometheus Jobs** (95+ available):
- Node Exporter: `node-exporter`, `aws-node-exporter`
- Database: `db-*` jobs for MySQL instances
- Application: Various service-specific jobs

**MySQL Servers** (61 available):
- DevOps DB: `aws-luckyus-devops-rw` (service topology)
- All databases accessible for dependency checks

**Redis Clusters** (74 available):
- Cache dependency verification

**CloudWatch**:
- EC2 metrics: `AWS/EC2` namespace
- Full log access for system and application logs

---

## Investigation Protocol

### Phase 0: Alert Validation

**Objective**: Extract and validate alert metadata

**Actions**:
1. Parse the alert payload to extract:
   - `alertname`: The specific alert type
   - `instance`: IP address or instance identifier (format: `ip:port`)
   - `severity`: Alert level (critical, warning, info)
   - `service_name`: Associated service name
   - `job`: Prometheus job name
   - `startsAt`: Alert start timestamp

2. Validate alert is for EC2 (not RDS, K8s, etc.)

3. Determine service priority level by querying DevOps database:
```sql
-- Query service level from DevOps DB
-- Server: aws-luckyus-devops-rw
SELECT service_name, level, owner, oncall_group
FROM service_registry
WHERE service_name = '<extracted_service_name>';
```

**Priority Classification**:
| Priority | Response Time | Examples |
|----------|---------------|----------|
| L0 | < 15 minutes | Payment, Order Processing, User Auth |
| L1 | < 30 minutes | Inventory, Notifications, Reporting |
| L2 | < 2 hours | Background Jobs, Analytics, Dev/Test |

---

### Phase 1: Data Availability Check

**Objective**: Verify monitoring data sources are accessible

**Actions**:

1. **Check Prometheus Connectivity**:
```promql
-- Use Grafana MCP: mcp__grafana__query_prometheus
-- Datasource UID: df8o21agxtkw0d (UMBQuerier-Luckin)
up{instance="<instance_ip>:9100"}
```

2. **Verify Node Exporter Status**:
```promql
node_exporter_build_info{instance="<instance_ip>:9100"}
```

3. **Check Data Freshness** (last scrape should be < 2 minutes):
```promql
time() - node_time_seconds{instance="<instance_ip>:9100"}
```

**If data unavailable**: Document gap and proceed with CloudWatch as fallback.

---

### Phase 2: Cross-System Health Assessment

**Objective**: Collect comprehensive system metrics

**Execute ALL queries in parallel using mcp__grafana__query_prometheus**:

#### 2a. Disk Metrics
```promql
# Disk Usage Percentage
100 - ((node_filesystem_avail_bytes{instance="<ip>:9100",fstype!="tmpfs"} / node_filesystem_size_bytes{instance="<ip>:9100",fstype!="tmpfs"}) * 100)

# Inode Usage
100 - ((node_filesystem_files_free{instance="<ip>:9100"} / node_filesystem_files{instance="<ip>:9100"}) * 100)

# Disk I/O Rate (5m average)
rate(node_disk_read_bytes_total{instance="<ip>:9100"}[5m])
rate(node_disk_written_bytes_total{instance="<ip>:9100"}[5m])

# I/O Wait
rate(node_cpu_seconds_total{instance="<ip>:9100",mode="iowait"}[5m]) * 100
```

#### 2b. Memory Metrics
```promql
# Memory Usage Percentage
100 - ((node_memory_MemAvailable_bytes{instance="<ip>:9100"} / node_memory_MemTotal_bytes{instance="<ip>:9100"}) * 100)

# Memory Components
node_memory_MemTotal_bytes{instance="<ip>:9100"}
node_memory_MemAvailable_bytes{instance="<ip>:9100"}
node_memory_Buffers_bytes{instance="<ip>:9100"}
node_memory_Cached_bytes{instance="<ip>:9100"}

# Swap Usage
(node_memory_SwapTotal_bytes{instance="<ip>:9100"} - node_memory_SwapFree_bytes{instance="<ip>:9100"}) / node_memory_SwapTotal_bytes{instance="<ip>:9100"} * 100
```

#### 2c. CPU Metrics
```promql
# CPU Usage (all modes)
100 - (avg(rate(node_cpu_seconds_total{instance="<ip>:9100",mode="idle"}[5m])) * 100)

# CPU by Mode
rate(node_cpu_seconds_total{instance="<ip>:9100"}[5m])

# Load Average
node_load1{instance="<ip>:9100"}
node_load5{instance="<ip>:9100"}
node_load15{instance="<ip>:9100"}

# CPU Count (for load average context)
count(node_cpu_seconds_total{instance="<ip>:9100",mode="idle"})
```

#### 2d. Network Metrics
```promql
# Network Throughput
rate(node_network_receive_bytes_total{instance="<ip>:9100",device!="lo"}[5m])
rate(node_network_transmit_bytes_total{instance="<ip>:9100",device!="lo"}[5m])

# Network Errors
rate(node_network_receive_errs_total{instance="<ip>:9100"}[5m])
rate(node_network_transmit_errs_total{instance="<ip>:9100"}[5m])

# TCP Connections
node_netstat_Tcp_CurrEstab{instance="<ip>:9100"}
node_sockstat_TCP_tw{instance="<ip>:9100"}
```

---

### Phase 3: Sibling Instance Analysis

**Objective**: Compare affected instance with peer instances in same service group

**Actions**:

1. **Identify sibling instances**:
```promql
# Find all instances with same job/service
up{job="<job_name>"}
```

2. **Compare key metrics across siblings**:
```promql
# Memory comparison
100 - ((node_memory_MemAvailable_bytes{job="<job_name>"} / node_memory_MemTotal_bytes{job="<job_name>"}) * 100)

# CPU comparison
100 - (avg by (instance) (rate(node_cpu_seconds_total{job="<job_name>",mode="idle"}[5m])) * 100)

# Disk comparison
100 - ((node_filesystem_avail_bytes{job="<job_name>",mountpoint="/"} / node_filesystem_size_bytes{job="<job_name>",mountpoint="/"}) * 100)
```

3. **Calculate anomaly score**: If affected instance > 1.5 standard deviations from mean, flag as anomalous.

---

### Phase 4: Service-Type Specific Investigation

**Objective**: Gather application-specific metrics based on service type

#### For Java Applications:
```promql
# JVM Heap Usage
jvm_memory_used_bytes{instance="<ip>:9100",area="heap"}
jvm_memory_max_bytes{instance="<ip>:9100",area="heap"}

# GC Activity
rate(jvm_gc_collection_seconds_sum{instance="<ip>:9100"}[5m])
rate(jvm_gc_collection_seconds_count{instance="<ip>:9100"}[5m])

# Thread Count
jvm_threads_current{instance="<ip>:9100"}
jvm_threads_peak{instance="<ip>:9100"}
```

#### For Process Stats:
```promql
# Process Count
node_procs_running{instance="<ip>:9100"}
node_procs_blocked{instance="<ip>:9100"}

# File Descriptors
process_open_fds{instance="<ip>:9100"}
process_max_fds{instance="<ip>:9100"}
```

---

### Phase 5: Database/Cache Correlation

**Objective**: Check related RDS and Redis dependencies

**Actions**:

1. **Identify database dependencies** from DevOps DB:
```sql
-- Server: aws-luckyus-devops-rw
SELECT db_host, db_name, connection_pool_size
FROM service_db_mapping
WHERE service_name = '<service_name>';
```

2. **Check Redis cache status**:
```
-- Use mcp__mcp-db-gateway__redis_command
-- Check memory and connections on relevant Redis cluster
INFO memory
INFO clients
INFO stats
```

3. **Query Redis metrics from Prometheus**:
```promql
-- Datasource UID: ff6p0gjt24phce (prometheus_redis)
redis_connected_clients{instance=~"<redis_cluster>.*"}
redis_memory_used_bytes{instance=~"<redis_cluster>.*"}
redis_commands_processed_total{instance=~"<redis_cluster>.*"}
```

---

### Phase 6: CloudWatch Integration

**Objective**: Cross-reference with AWS CloudWatch metrics and logs

**Actions**:

1. **Get CloudWatch EC2 Metrics** (use mcp__cloudwatch-server__get_metric_data):
```
Namespace: AWS/EC2
Metrics: CPUUtilization, NetworkIn, NetworkOut, DiskReadOps, DiskWriteOps
Dimensions: InstanceId=<instance_id>
Period: 300 seconds
```

2. **Query CloudWatch Logs** (use mcp__cloudwatch-server__execute_log_insights_query):
```
# Search for errors in system logs
fields @timestamp, @message
| filter @message like /error|Error|ERROR|exception|Exception/
| sort @timestamp desc
| limit 50
```

3. **Check for recent alarms**:
Use mcp__cloudwatch-server__get_active_alarms to find related alarms.

---

### Phase 7: Output Report Generation

**Objective**: Generate structured investigation report

**Report Template**:

```markdown
## EC2 Investigation Report

### Alert Summary
| Field | Value |
|-------|-------|
| Alert Name | [alertname] |
| Instance | [instance_ip] |
| Instance ID | [aws_instance_id] |
| Service | [service_name] |
| Severity | [L0/L1/L2] |
| Start Time | [timestamp] |
| Duration | [duration] |

### Service Context
- **Service Level**: [L0/L1/L2]
- **Service Owner**: [owner]
- **On-Call Group**: [oncall_group]

### Investigation Findings

#### Phase 1: Data Availability
- Prometheus: [Available/Unavailable]
- Node Exporter: [Running/Down]
- Data Freshness: [X seconds]

#### Phase 2: System Health
| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| Disk Usage | [X%] | 85% | [OK/WARNING/CRITICAL] |
| Memory Usage | [X%] | 90% | [OK/WARNING/CRITICAL] |
| CPU Usage | [X%] | 80% | [OK/WARNING/CRITICAL] |
| I/O Wait | [X%] | 20% | [OK/WARNING/CRITICAL] |
| Load Average | [X] | [cores*2] | [OK/WARNING/CRITICAL] |

#### Phase 3: Sibling Comparison
| Instance | Memory | CPU | Disk | Status |
|----------|--------|-----|------|--------|
| [ip1] | [X%] | [X%] | [X%] | [Normal/Anomalous] |
| [ip2] | [X%] | [X%] | [X%] | [Normal/Anomalous] |

**Anomaly Detection**: [Affected instance is/is not anomalous compared to siblings]

#### Phase 4: Application Metrics
- [Application-specific findings]

#### Phase 5: Dependencies
- **Database Status**: [Healthy/Issues detected]
- **Cache Status**: [Healthy/Issues detected]
- **Connection Pools**: [Adequate/Exhausted]

#### Phase 6: CloudWatch Correlation
- [CloudWatch findings and correlations]

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
<summary>CloudWatch Data</summary>

[Include CloudWatch metrics and logs]

</details>

---
*Investigation completed at [timestamp]*
*Skill Version: EC2 Alert Investigation SOP v4.0*
```

---

## MCP Tools Reference

| Tool | Server | Purpose |
|------|--------|---------|
| `mcp__grafana__query_prometheus` | grafana | Primary metrics queries |
| `mcp__grafana__list_prometheus_label_values` | grafana | Discover instance labels |
| `mcp__mcp-db-gateway__mysql_query` | mcp-db-gateway | Service topology lookup |
| `mcp__mcp-db-gateway__redis_command` | mcp-db-gateway | Cache correlation |
| `mcp__cloudwatch-server__get_metric_data` | cloudwatch-server | AWS metrics |
| `mcp__cloudwatch-server__execute_log_insights_query` | cloudwatch-server | CloudWatch logs |
| `mcp__cloudwatch-server__get_active_alarms` | cloudwatch-server | Related alarms |

## Key Datasource UIDs

| Datasource | UID | Use |
|------------|-----|-----|
| UMBQuerier-Luckin | `df8o21agxtkw0d` | Primary EC2/Infrastructure metrics |
| prometheus | `ff7hkeec6c9a8e` | General metrics |
| prometheus_redis | `ff6p0gjt24phce` | Redis metrics |

## DevOps Database

| Server | Purpose |
|--------|---------|
| `aws-luckyus-devops-rw` | Service registry, alert logs, topology |

---

## Example Usage

When investigating an EC2 alert, invoke this skill with the alert payload:

```
/investigate-ec2 {
  "alertname": "MemUsedPercent",
  "instance": "10.1.2.3:9100",
  "severity": "critical",
  "service_name": "order-service",
  "job": "order-nodes"
}
```

The skill will execute all 8 phases automatically and generate a comprehensive report.
