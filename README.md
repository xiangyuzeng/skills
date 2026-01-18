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
```

**Option 2: Natural Language**
```
"Investigate this EC2 alert: [paste alert JSON]"
"Help me diagnose this RDS connection issue: [paste alert]"
"Why is my pod crashing? [paste K8s alert]"
```

---

## Available Skills

| Skill | File | Version | Use Case |
|-------|------|---------|----------|
| EC2 Alert Investigation | `ec2-alert-investigation.md` | v4.1 | EC2 instance issues (disk, memory, CPU, network) |
| RDS Alert Investigation | `rds-alert-investigation.md` | v2.0 | Database issues (connections, slow queries, replication) |
| K8s Alert Investigation | `k8s-alert-investigation.md` | v2.0 | Kubernetes/EKS issues (pods, nodes, deployments) |

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
├── README.md                      # This file
├── ec2-alert-investigation.md     # EC2 skill (406 lines)
├── rds-alert-investigation.md     # RDS skill (667 lines)
├── k8s-alert-investigation.md     # K8s skill (776 lines)
├── skills-comparison.md           # Side-by-side comparison
└── use-cases.md                   # Real-world use case examples
```

---

*Version: 1.0 | Last Updated: 2026-01-18*
