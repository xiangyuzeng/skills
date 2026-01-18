# Skills Comparison Matrix

> Side-by-side comparison of all three alert investigation skills

## Overview

| Feature | EC2 Skill | RDS Skill | K8s Skill |
|---------|-----------|-----------|-----------|
| **Version** | v4.1 | v2.0 | v2.0 |
| **Lines of Code** | ~430 | ~690 | ~850 |
| **Investigation Phases** | 8 | 8 | 9 |
| **Primary Target** | EC2 Instances | RDS Databases | EKS Pods/Nodes |
| **Primary Data Source** | Prometheus | MySQL/CloudWatch | Prometheus (kube-state-metrics) |

---

## Alert Types Handled

### EC2 Skill
| Alert Type | Prometheus Metric | Threshold |
|------------|-------------------|-----------|
| Disk Usage | `node_filesystem_avail_bytes` | > 85% |
| Disk Inodes | `node_filesystem_files_free` | > 90% |
| Memory | `node_memory_MemAvailable_bytes` | > 90% |
| CPU | `node_cpu_seconds_total` | > 80% |
| I/O Wait | `node_cpu_seconds_total{mode="iowait"}` | > 20% |
| Network Errors | `node_network_*_errs_total` | > 0 |

### RDS Skill
| Alert Type | CloudWatch Metric | Threshold |
|------------|-------------------|-----------|
| Connections | `DatabaseConnections` | > 80% of max |
| Storage | `FreeStorageSpace` | < 20% |
| CPU | `CPUUtilization` | > 80% |
| Replication Lag | `ReplicaLag` | > 60s |
| Slow Queries | CloudWatch Logs | > baseline |
| IOPS | `ReadIOPS`, `WriteIOPS` | > provisioned |

### K8s Skill
| Alert Type | Source | Condition |
|------------|--------|-----------|
| OOMKilled | K8s Events | `reason=OOMKilled` |
| CrashLoopBackOff | Pod Status | `waiting.reason=CrashLoopBackOff` |
| Pending | Pod Status | `phase=Pending` > 5m |
| Node NotReady | Node Conditions | `Ready=False` |
| Replicas Mismatch | Deployment | `spec.replicas != status.availableReplicas` |
| HPA Maxed Out | HPA Status | `currentReplicas = maxReplicas` |

---

## MCP Tools Usage

| MCP Tool | EC2 | RDS | K8s |
|----------|-----|-----|-----|
| `mcp__grafana__query_prometheus` | ✅ Primary | ✅ Primary | ✅ Secondary |
| `mcp__mcp-db-gateway__mysql_query` | ✅ Topology | ✅ Direct Query | ✅ Topology |
| `mcp__mcp-db-gateway__redis_command` | ✅ Cache Check | ❌ | ❌ |
| `mcp__cloudwatch-server__get_metric_data` | ✅ AWS Metrics | ✅ RDS Metrics | ✅ Container Insights |
| `mcp__cloudwatch-server__execute_log_insights_query` | ✅ System Logs | ✅ Slow Query Logs | ✅ Container Logs |
| `mcp__cloudwatch-server__get_active_alarms` | ✅ | ✅ | ✅ |
| `mcp__eks-server__list_k8s_resources` | ❌ | ❌ | ✅ Primary |
| `mcp__eks-server__get_pod_logs` | ❌ | ❌ | ✅ Primary |
| `mcp__eks-server__get_k8s_events` | ❌ | ❌ | ✅ Primary |

---

## Data Sources by Skill

### EC2 Skill Data Sources
| Source | UID/Server | Purpose |
|--------|------------|---------|
| UMBQuerier-Luckin | `df8o21agxtkw0d` | Node Exporter metrics |
| prometheus | `ff7hkeec6c9a8e` | General metrics |
| prometheus_redis | `ff6p0gjt24phce` | Redis cache metrics |
| DevOps MySQL | `aws-luckyus-devops-rw` | Service topology |
| CloudWatch | AWS/EC2 | Instance metrics |

### RDS Skill Data Sources
| Source | Purpose |
|--------|---------|
| 61 MySQL Servers | Direct database queries |
| 3 PostgreSQL Servers | Analytics queries |
| 52 RDS Slow Query Log Groups | Slow query analysis |
| CloudWatch AWS/RDS | Database metrics |
| DevOps MySQL | Service-DB mappings |

### K8s Skill Data Sources
| Source | Purpose |
|--------|---------|
| EKS Cluster API | Pod, Node, Deployment status |
| Container Insights | Container CPU/memory metrics |
| CloudWatch Logs | Container application logs |
| Prometheus | Kubernetes metrics |
| DevOps MySQL | Service topology |

---

## Investigation Phase Comparison

| Phase | EC2 | RDS | K8s |
|-------|-----|-----|-----|
| 0 | Alert Validation | Alert Validation | Alert Validation |
| 1 | Data Availability | Data Availability | Data Availability |
| 2 | System Health | Database Health | Pod Health |
| 3 | Sibling Analysis | Slow Query Analysis | Node Health |
| 4 | Service-Type Specific | Replication Status | Workload Analysis |
| 5 | DB/Cache Correlation | Application Correlation | Container Logs |
| 6 | CloudWatch Integration | CloudWatch Deep Dive | Resource Utilization |
| 7 | Report Generation | Report Generation | Network Analysis |
| 8 | - | - | Report Generation |

---

## Unique Features by Skill

### EC2 Skill Unique Features
- **Sibling Instance Comparison**: Compares affected instance with peers running the same service
- **Anomaly Detection**: Calculates if instance is > 1.5 standard deviations from mean
- **JVM Metrics**: Special handling for Java applications (heap, GC, threads)
- **Process Stats**: Process count, file descriptor monitoring
- **Redis Cache Correlation**: Checks cache health for affected services

### RDS Skill Unique Features
- **CloudWatch Slow Query Log Analysis**: Queries 52 RDS slow query log groups
- **Performance Insights**: Deep query analysis when available
- **Replication Topology**: Master/replica relationship analysis
- **Connection Pool Analysis**: Tracks connection sources by application
- **InnoDB Status**: Direct `SHOW ENGINE INNODB STATUS` for deadlock analysis
- **Process List Analysis**: Identifies blocking queries

### K8s Skill Unique Features
- **Multi-Container Support**: Handles pods with multiple containers
- **Event Timeline**: Kubernetes event analysis with timestamps
- **Resource Quota Analysis**: Namespace resource limits vs usage
- **Network Policy Analysis**: Service mesh and network policy impact
- **HPA Analysis**: Horizontal Pod Autoscaler status and history
- **Node Affinity/Taint Analysis**: Scheduling constraint troubleshooting

---

## Report Output Comparison

| Report Section | EC2 | RDS | K8s |
|----------------|-----|-----|-----|
| Alert Summary | ✅ | ✅ | ✅ |
| Service Context | ✅ | ✅ | ✅ |
| Health Metrics Table | ✅ | ✅ | ✅ |
| Sibling Comparison | ✅ | ❌ | ❌ |
| Slow Query Analysis | ❌ | ✅ | ❌ |
| Container Status | ❌ | ❌ | ✅ |
| Root Cause Analysis | ✅ | ✅ | ✅ |
| Recommendations | ✅ | ✅ | ✅ |
| Evidence (Collapsible) | ✅ | ✅ | ✅ |

---

## When to Use Which Skill

| Scenario | Recommended Skill |
|----------|-------------------|
| VM disk space alert | EC2 Skill |
| VM memory high | EC2 Skill |
| Database connection exhausted | RDS Skill |
| Slow database queries | RDS Skill |
| Replication lag | RDS Skill |
| Pod OOMKilled | K8s Skill |
| Pod CrashLoopBackOff | K8s Skill |
| Node NotReady | K8s Skill |
| Deployment not scaling | K8s Skill |
| Application on EC2 slow | EC2 + RDS (if DB involved) |
| Microservice in K8s slow | K8s + RDS (if DB involved) |

---

## Cross-Skill Dependencies

Some investigations may require multiple skills:

```
EC2 Alert → EC2 Skill → Check DB dependencies → RDS Skill
K8s Alert → K8s Skill → Check external DB → RDS Skill
RDS Alert → RDS Skill → Check client VMs → EC2 Skill
```

**Recommendation**: When an investigation reveals dependencies on other infrastructure types, chain the appropriate skill.

---

*Generated: 2026-01-18*
