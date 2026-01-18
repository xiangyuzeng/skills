# Alert Investigation Skills for Claude Code

> A comprehensive collection of alert investigation skills designed for automated infrastructure diagnosis using MCP (Model Context Protocol) data sources.

## Quick Start

These skills are designed to be invoked when you receive alerts from your monitoring systems. Each skill provides a structured, multi-phase investigation workflow that automatically queries multiple data sources to diagnose issues.

### How to Invoke Skills

**Option 1: Direct Invocation**
```
/investigate-ec2 <alert_payload>
/investigate-rds <alert_payload>
/investigate-k8s <alert_payload>
/investigate-redis <alert_payload>
/investigate-elasticsearch <alert_payload>
/investigate-apm <alert_payload>
```

**Option 2: Natural Language**
```
"Investigate this EC2 alert: [paste alert JSON]"
"Help me diagnose this RDS connection issue: [paste alert]"
"Why is my pod crashing? [paste K8s alert]"
"Redis memory is high, please investigate"
"Elasticsearch cluster is yellow, help me diagnose"
"Service response time is high, investigate the APM alert"
```

---

## Available Skills

| Skill | File | Version | Use Case |
|-------|------|---------|----------|
| EC2 Alert Investigation | `ec2-alert-investigation.md` | v5.0 | EC2/VM instance issues (disk, memory, CPU, network, I/O) |
| RDS Alert Investigation | `rds-alert-investigation.md` | v2.0 | Database issues (connections, slow queries, replication) |
| K8s Alert Investigation | `k8s-alert-investigation.md` | v2.0 | Kubernetes/EKS issues (pods, nodes, deployments, OOMKilled) |
| Redis Alert Investigation | `redis-alert-investigation.md` | v1.0 | Redis/ElastiCache issues (memory, CPU, connections, latency) |
| Elasticsearch Alert Investigation | `elasticsearch-alert-investigation.md` | v1.0 | OpenSearch/ES issues (cluster health, CPU, disk, shards) |
| APM Alert Investigation | `apm-alert-investigation.md` | v1.0 | Application issues (JVM, response time, errors, GC) |

---

## Skill Details

### 1. EC2 Alert Investigation Skill

**Triggers:** Use this skill when you receive alerts related to EC2 instances:
- `DiskUsedPercent`, `DiskInodesUsedPercent` - Disk space/inode exhaustion
- `MemUsedPercent` - Memory pressure, OOM events
- `CPUTotalUsedPercent` - CPU throttling, high load
- `DiskIOReadBytes`, `DiskIOWriteBytes` - I/O bottlenecks
- `NetBytesIn`, `NetBytesOut` - Network issues

**Investigation Phases:**
1. **Alert Validation** - Parse metadata, validate EC2 target
2. **Data Availability Check** - Verify Prometheus/Node Exporter connectivity
3. **Cross-System Health Assessment** - Disk, memory, CPU, network metrics
4. **Sibling Instance Analysis** - Compare with peer instances
5. **Service-Type Specific** - JVM, process-level metrics
6. **Database/Cache Correlation** - Check RDS/Redis dependencies
7. **CloudWatch Integration** - AWS metrics and logs
8. **Report Generation** - Structured investigation report

**Example:**
```json
{
  "alertname": "DiskUsedPercent",
  "instance": "10.1.2.3:9100",
  "severity": "critical",
  "service_name": "order-service",
  "job": "order-nodes"
}
```

**MCP Tools Used:**
- `mcp__grafana__query_prometheus` (Datasource: `df8o21agxtkw0d`)
- `mcp__mcp-db-gateway__mysql_query` (Server: `aws-luckyus-devops-rw`)
- `mcp__cloudwatch-server__get_metric_data`
- `mcp__cloudwatch-server__execute_log_insights_query`

---

### 2. RDS Alert Investigation Skill

**Triggers:** Use this skill when you receive alerts related to RDS databases:
- `RDSConnections`, `ConnectionExhausted` - Connection pool issues
- `SlowQueryCount`, `QueryLatency` - Slow query problems
- `FreeStorageSpace`, `DiskQueueDepth` - Storage issues
- `ReplicationLag`, `ReplicaLag` - Replication problems
- `CPUUtilization`, `DatabaseMemoryUsage` - Resource exhaustion

**Investigation Phases:**
1. **Alert Validation** - Parse metadata, identify database instance
2. **Data Availability Check** - Verify monitoring connectivity
3. **Database Health Assessment** - Connection, query, storage metrics
4. **Slow Query Analysis** - CloudWatch slow query logs
5. **Replication Status** - Master/replica lag analysis
6. **Application Correlation** - Identify dependent services
7. **CloudWatch Deep Dive** - Enhanced monitoring metrics
8. **Report Generation** - Comprehensive diagnosis report

**Example:**
```json
{
  "alertname": "RDSConnectionHigh",
  "instance": "aws-luckyus-order-rw",
  "severity": "warning",
  "current_connections": 450,
  "max_connections": 500
}
```

**MCP Tools Used:**
- `mcp__grafana__query_prometheus` (Datasource: `df8o21agxtkw0d`)
- `mcp__mcp-db-gateway__mysql_query` (Servers: 61 MySQL, 3 PostgreSQL)
- `mcp__cloudwatch-server__execute_log_insights_query` (52 RDS slow query log groups)
- `mcp__cloudwatch-server__get_metric_data` (Namespace: AWS/RDS)

---

### 3. K8s (EKS) Alert Investigation Skill

**Triggers:** Use this skill when you receive alerts related to Kubernetes:
- `KubePodCrashLooping`, `OOMKilled` - Pod crash issues
- `KubePodNotReady`, `PodPending` - Pod scheduling problems
- `KubeNodeNotReady`, `NodePressure` - Node health issues
- `KubeDeploymentReplicasMismatch` - Deployment issues
- `KubeHpaMaxedOut`, `KubeHpaReplicasMismatch` - HPA problems

**Investigation Phases:**
1. **Alert Validation** - Parse K8s-specific metadata
2. **Data Availability Check** - Verify EKS and Prometheus access
3. **Pod Health Assessment** - Container status, restarts, OOM
4. **Node Health Assessment** - Node conditions, resource pressure
5. **Workload Analysis** - Deployment, ReplicaSet, HPA status
6. **Container Logs** - Pod and CloudWatch container logs
7. **Resource Utilization** - CPU, memory requests vs limits
8. **Network Analysis** - Service, ingress, network policies
9. **Report Generation** - Comprehensive K8s diagnosis

**Example:**
```json
{
  "alertname": "KubePodCrashLooping",
  "namespace": "production",
  "pod": "order-service-abc123",
  "container": "order-api",
  "severity": "critical"
}
```

**MCP Tools Used:**
- `mcp__eks-server__list_k8s_resources`
- `mcp__eks-server__get_pod_logs`
- `mcp__eks-server__get_k8s_events`
- `mcp__grafana__query_prometheus` (Container Insights)
- `mcp__cloudwatch-server__execute_log_insights_query`

---

### 4. Redis Alert Investigation Skill

**Triggers:** Use this skill when you receive alerts related to Redis/ElastiCache:
- `Redis CPU使用率大于90%` - High CPU utilization
- `Redis 内存使用率持续3分钟超过70%` - Memory pressure, key eviction
- `Redis 实例连接数使用率大于30%` - Connection pool exhaustion
- `Redis 实例命令平均时延大于2ms` - Latency issues
- `Redis 实例流量大于32Mbps` - Network throughput issues

**Investigation Phases:**
1. **Alert Validation** - Parse metadata, identify Redis cluster
2. **Data Availability Check** - Verify Prometheus Redis datasource
3. **Redis Health Assessment** - Memory, CPU, connections, latency metrics
4. **Direct Redis Diagnostics** - INFO, SLOWLOG, CLIENT LIST commands
5. **Service Dependency Analysis** - Identify applications using this cache
6. **AWS CloudWatch Integration** - ElastiCache metrics
7. **Report Generation** - Comprehensive Redis diagnosis

**Example:**
```json
{
  "alertname": "Redis 内存使用率持续3分钟超过70%",
  "instance": "luckyus-isales-order",
  "severity": "warning",
  "current_value": 75
}
```

**MCP Tools Used:**
- `mcp__grafana__query_prometheus` (Datasource: `ff6p0gjt24phce` - prometheus_redis)
- `mcp__mcp-db-gateway__redis_command` (74 clusters available)
- `mcp__cloudwatch-server__get_metric_data` (Namespace: AWS/ElastiCache)

---

### 5. Elasticsearch/OpenSearch Alert Investigation Skill

**Triggers:** Use this skill when you receive alerts related to Elasticsearch/OpenSearch:
- `AWS-ES CPU 使用率大于90%` - High CPU utilization
- `AWS-ES 集群状态Red` - Critical cluster health (primary shards unallocated)
- `AWS-ES 集群状态Yellow` - Degraded cluster health (replica shards missing)
- `AWS-ES磁盘空间不足10G` - Storage capacity critical
- `JVMMemoryPressure` - JVM heap memory pressure

**Investigation Phases:**
1. **Alert Validation** - Parse metadata, identify ES domain
2. **Data Availability Check** - Verify CloudWatch access
3. **Cluster Health Assessment** - Status, CPU, memory, shards, nodes
4. **Root Cause Analysis by Alert Type** - Specific investigation per alert
5. **Service Dependency Analysis** - Applications using this ES cluster
6. **Report Generation** - Comprehensive ES diagnosis

**Example:**
```json
{
  "alertname": "AWS-ES 集群状态Red",
  "domain": "luckyus-logs-es",
  "severity": "critical"
}
```

**MCP Tools Used:**
- `mcp__cloudwatch-server__get_metric_data` (Namespace: AWS/ES)
- `mcp__grafana__query_prometheus` (Datasource: `df8o21agxtkw0d`)
- `mcp__mcp-db-gateway__mysql_query` (Service topology)

---

### 6. APM (Application Performance Monitoring) Alert Investigation Skill

**Triggers:** Use this skill when you receive alerts related to application performance:
- `【iZeus-策略X】` - iZeus application monitoring alerts (response time, error rate)
- `FGC次数大于1`, `YGC次数大于15` - JVM garbage collection alerts
- `服务响应时间` - Service response time degradation
- `服务异常数` - Service error rate increase
- `服务调用数突然下降或上升` - Traffic anomalies

**Investigation Phases:**
1. **Alert Validation** - Parse metadata, identify service
2. **Data Availability Check** - Verify iZeus/Prometheus connectivity
3. **JVM Health Assessment** - Heap, GC, threads, class loading
4. **Response Time Analysis** - P50, P90, P99 latencies
5. **Error Rate Investigation** - Exception patterns, error types
6. **Traffic Analysis** - Request volume, throughput changes
7. **Dependency Check** - Database, Redis, external services
8. **CloudWatch Log Analysis** - Exception search and patterns
9. **Report Generation** - Comprehensive APM diagnosis

**Example:**
```json
{
  "alertname": "【iZeus-策略5】服务响应时间增加",
  "service_name": "order-api",
  "severity": "warning",
  "response_time_p99": 2500
}
```

**MCP Tools Used:**
- `mcp__grafana__query_prometheus` (Datasource: `df8o21agxtkw0d`)
- `mcp__cloudwatch-server__execute_log_insights_query`
- `mcp__mcp-db-gateway__mysql_query` (Service dependencies)
- `mcp__mcp-db-gateway__redis_command` (Cache health)

---

## Service Priority Classification

All skills use a consistent priority classification:

| Priority | Response Time | Examples |
|----------|---------------|----------|
| **L0** | < 15 minutes | Payment, Order Processing, User Auth |
| **L1** | < 30 minutes | Inventory, Notifications, Reporting |
| **L2** | < 2 hours | Background Jobs, Analytics, Dev/Test |

Service levels are queried from the DevOps database:
```sql
SELECT service_name, level, owner, oncall_group
FROM service_registry
WHERE service_name = '<service_name>';
```

---

## Verified Data Sources (Tested 2026-01-18)

| Source | MCP Server | Count/Details | Primary Use |
|--------|------------|---------------|-------------|
| Prometheus | `grafana` | 3 datasources, 95+ jobs | Real-time metrics |
| MySQL | `mcp-db-gateway` | 61 servers | Service topology, direct queries |
| PostgreSQL | `mcp-db-gateway` | 3 servers | Audit logs, analytics |
| Redis | `mcp-db-gateway` | 74 clusters | Cache correlation |
| CloudWatch Logs | `cloudwatch-server` | 64 RDS slow query logs | Slow query analysis |
| CloudWatch Metrics | `cloudwatch-server` | Full access | AWS service metrics |
| EKS | Prometheus (kube-state-metrics) | 40+ namespaces | Kubernetes metrics |

### Key Prometheus Datasources
| Name | UID | Purpose |
|------|-----|---------|
| UMBQuerier-Luckin | `df8o21agxtkw0d` | PRIMARY - Node and K8s metrics |
| prometheus | `ff7hkeec6c9a8e` | General metrics |
| prometheus_redis | `ff6p0gjt24phce` | Redis metrics (74 clusters) |

### Key Database Servers
- **DevOps**: `aws-luckyus-devops-rw` (service registry)
- **Core**: `aws-luckyus-salesorder-rw`, `aws-luckyus-payment-rw`
- **EKS Cluster**: `prod-worker01-eks-us`

---

## Output Format

All skills generate a standardized investigation report with:

1. **Alert Summary** - Alert metadata and context
2. **Service Context** - Priority level, owner, on-call group
3. **Investigation Findings** - Phase-by-phase results with metrics tables
4. **Root Cause Analysis** - Primary cause and contributing factors
5. **Recommendations** - Prioritized actions (P0/P1/P2)
6. **Evidence** - Collapsed sections with query results and logs

---

## Best Practices

1. **Always provide the full alert payload** - Include all available labels and annotations
2. **Specify the service name** - Helps with topology lookup and sibling comparison
3. **Include timestamps** - Helps narrow the investigation window
4. **Mention related incidents** - Helps correlate with other ongoing issues

---

## Troubleshooting

**Q: Prometheus data unavailable?**
A: Skills will automatically fall back to CloudWatch metrics. Check that Node Exporter is running on the target instance.

**Q: DevOps database connection failed?**
A: Service topology information will be skipped. The investigation will continue with available data.

**Q: CloudWatch access denied?**
A: Ensure the IAM role has `CloudWatchLogsReadOnlyAccess` permission.

---

## File Structure

```
/app/skills/
├── README.md                              # This file
├── ec2-alert-investigation.md             # EC2/VM skill (v5.0)
├── rds-alert-investigation.md             # RDS skill (v2.0)
├── k8s-alert-investigation.md             # K8s/EKS skill (v2.0)
├── redis-alert-investigation.md           # Redis/ElastiCache skill (v1.0)
├── elasticsearch-alert-investigation.md   # Elasticsearch/OpenSearch skill (v1.0)
├── apm-alert-investigation.md             # APM/Application skill (v1.0)
├── skills-comparison.md                   # Side-by-side comparison
└── use-cases.md                           # Real-world use case examples
```

---

## Alert Coverage Summary

| Alert Category | Skill | Alert Examples |
|---------------|-------|----------------|
| **VM/EC2** | EC2 Investigation | 【vm-CPU】, 【vm-内存】, 【vm-磁盘】, 【vm-io】, 【vm-tcp】 |
| **Database** | RDS Investigation | DatabaseConnections, SlowQueries, ReplicaLag, FreeStorageSpace |
| **Kubernetes** | K8s Investigation | KubePodCrashLooping, OOMKilled, Pending, NodeNotReady |
| **Cache** | Redis Investigation | Redis CPU, 内存使用率, 连接数, 命令时延 |
| **Search** | Elasticsearch Investigation | AWS-ES CPU, 集群状态Red/Yellow, 磁盘空间不足 |
| **Application** | APM Investigation | 【iZeus-策略X】, FGC, YGC, 服务响应时间, 异常数 |

---

*Version: 2.0 | Last Updated: 2026-01-18*
