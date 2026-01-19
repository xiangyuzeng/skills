# Alert Investigation Skills for Claude Code

## Complete Documentation and Use Cases Guide

This documentation provides comprehensive guidance on using the 6 alert investigation skills designed for Claude Code with MCP (Model Context Protocol) data sources.

---

## Table of Contents

1. [Overview](#overview)
2. [Quick Start](#quick-start)
3. [Skills Summary](#skills-summary)
4. [Detailed Use Cases](#detailed-use-cases)
5. [Data Sources Reference](#data-sources-reference)
6. [Troubleshooting Workflows](#troubleshooting-workflows)
7. [Best Practices](#best-practices)
8. [Integration Guide](#integration-guide)

---

## Overview

### What Are These Skills?

These skills are specialized investigation protocols that enable Claude Code to systematically diagnose infrastructure alerts across your AWS environment. Each skill follows a multi-phase methodology:

```
Alert Received → Validation → Data Collection → Analysis → Root Cause → Recommendations
```

### Key Features

| Feature | Description |
|---------|-------------|
| **Parallel Execution** | Queries multiple data sources simultaneously |
| **Cross-Skill Integration** | Skills suggest related investigations when issues span domains |
| **Error Handling** | Fallback mechanisms when data sources are unavailable |
| **Priority Classification** | L0/L1/L2 service tiers for response SLAs |
| **Evidence Documentation** | Structured reports with supporting data |

### Service Level Agreements

| Priority | Services | Response Time | Examples |
|----------|----------|---------------|----------|
| **L0** | Critical business | 5-15 minutes | Payment, Order, Auth |
| **L1** | Core services | 15-30 minutes | Inventory, Gateway, User |
| **L2** | Supporting | 30min-2 hours | Analytics, Batch, Logging |

---

## Quick Start

### Invocation Methods

**1. Slash Commands (Recommended)**
```
/investigate-ec2 i-0abc123def456
/investigate-rds order_db connection-exhaustion
/investigate-k8s production/payment-service OOMKilled
/investigate-redis session-cache memory
/investigate-elasticsearch search-cluster health-red
/investigate-apm payment-service latency
```

**2. Natural Language**
```
"Investigate the EC2 alert for instance 10.0.1.50"
"Debug the RDS connection issues on payment_db"
"Why is the payment-service pod crashing?"
"Check Redis memory pressure on cart-cache"
```

**3. Automatic Activation**
Skills automatically trigger when you describe infrastructure issues matching their trigger conditions.

---

## Skills Summary

| Skill | Version | Target | Key Capabilities |
|-------|---------|--------|------------------|
| EC2 Investigation | v6.0 | VM/Instances | CPU, memory, disk, I/O, network |
| RDS Investigation | v3.0 | Databases | Connections, queries, replication |
| K8s Investigation | v3.0 | Kubernetes | Pods, nodes, deployments |
| Redis Investigation | v2.0 | Cache | Memory, latency, connections |
| Elasticsearch Investigation | v2.0 | Search | Cluster health, shards, JVM |
| APM Investigation | v2.0 | Applications | JVM, latency, errors |

---

## Detailed Use Cases

### 1. EC2 Alert Investigation

#### When to Use
- CPU utilization > 80% alerts
- Memory exhaustion warnings
- Disk space critical alerts
- High I/O wait times
- Network packet drops

#### Example Scenarios

**Scenario A: Disk Full Alert**
```
Alert: "disk_usage_critical" on prod-app-01
```

Investigation Flow:
1. Validate which filesystem is affected
2. Check disk usage trend over 24h
3. Identify large files/directories
4. Compare with sibling instances
5. Check if log rotation is working
6. Recommend cleanup or expansion

**Scenario B: High CPU with Normal Memory**
```
Alert: "cpu_high" on prod-worker-05
```

Investigation Flow:
1. Check per-process CPU usage
2. Identify runaway processes
3. Check for stuck threads
4. Compare workload with peers
5. Determine if scaling needed

#### Sample Output
```markdown
## EC2 Investigation Report

### Alert Summary
- **Instance**: i-0abc123 (10.0.1.50)
- **Service**: order-service (Priority: L0)
- **Alert**: disk_usage_critical
- **Duration**: 45 minutes

### Key Findings
1. /var/log filesystem at 94% (critical)
2. Application logs not rotated for 7 days
3. Sibling instance prod-app-02 at 67% (normal)

### Root Cause
Log rotation cron job failed on Jan 15 after package update

### Recommendations
1. **Immediate**: Clear old logs > 7 days
2. **Short-term**: Fix logrotate configuration
3. **Long-term**: Implement centralized logging
```

---

### 2. RDS Alert Investigation

#### When to Use
- Connection count > 80% limit
- Slow query alerts
- Replication lag warnings
- Storage space alerts
- CPU spike on database

#### Example Scenarios

**Scenario A: Connection Exhaustion**
```
Alert: "rds_connections_high" on order-db
```

Investigation Flow:
1. Check current vs max connections
2. Identify connection sources by user/host
3. Find idle connections holding slots
4. Check application connection pool settings
5. Look for connection leaks in recent deployments

**Scenario B: Replication Lag**
```
Alert: "replica_lag_critical" on order-db-replica
```

Investigation Flow:
1. Check Seconds_Behind_Master
2. Identify large transactions on primary
3. Check replica I/O and SQL thread status
4. Compare replica vs primary performance
5. Check for long-running queries blocking replication

---

### 3. Kubernetes/EKS Alert Investigation

#### When to Use
- OOMKilled events
- CrashLoopBackOff status
- Pending pods not scheduling
- Node NotReady conditions
- CPU throttling alerts

#### Example Scenarios

**Scenario A: OOMKilled Pod**
```
Alert: "pod_oom_killed" on payment-service-7d4f8
```

Investigation Flow:
1. Get pod events and termination reason
2. Check memory usage vs limits
3. Review memory trend before OOM
4. Check if memory leak pattern exists
5. Compare with other replicas
6. Review recent deployments for memory changes

**Scenario B: Pending Pod**
```
Alert: "pod_pending" on inventory-service
```

Investigation Flow:
1. Check pod events for scheduling failure reason
2. Verify node resources available
3. Check node selectors and tolerations
4. Review affinity/anti-affinity rules
5. Check for PVC binding issues

---

### 4. Redis Alert Investigation

#### When to Use
- Memory > 85% alerts
- High eviction rates
- Latency spikes (> 2ms)
- Blocked clients
- Replication issues

#### Example Scenarios

**Scenario A: Memory Pressure**
```
Alert: "redis_memory_high" on session-cache
```

Investigation Flow:
1. Check used_memory vs maxmemory
2. Analyze memory fragmentation ratio
3. Check eviction policy and rates
4. Identify big keys
5. Review TTL settings on keys
6. Check for memory leak patterns

**Scenario B: Slow Commands**
```
Alert: "redis_latency_high" on product-cache
```

Investigation Flow:
1. Run SLOWLOG GET 20
2. Identify O(N) operations on large keys
3. Check for KEYS commands in production
4. Review client connection patterns
5. Check network throughput limits

---

### 5. Elasticsearch Alert Investigation

#### When to Use
- Cluster status RED/YELLOW
- JVM heap > 85%
- Unassigned shards
- Query latency spikes
- Storage running low

#### Example Scenarios

**Scenario A: Cluster RED**
```
Alert: "es_cluster_red" on search-cluster
```

Investigation Flow:
1. Check which shards are unassigned
2. Identify reason for unassignment
3. Check node status and availability
4. Review disk space on data nodes
5. Check for recent node failures

**Scenario B: JVM Pressure**
```
Alert: "es_jvm_high" on search-cluster
```

Investigation Flow:
1. Check heap usage per node
2. Analyze GC frequency and duration
3. Look for circuit breaker trips
4. Check query patterns for heavy aggregations
5. Review field data and filter cache sizes

---

### 6. APM Alert Investigation

#### When to Use
- Response time > SLA
- Error rate spikes
- JVM CPU/memory alerts
- GC pause warnings
- Thread pool exhaustion

#### Example Scenarios

**Scenario A: Latency Spike**
```
Alert: "response_time_high" on payment-service
```

Investigation Flow:
1. Check p50/p95/p99 latency trends
2. Identify slow endpoints
3. Check downstream dependency latency
4. Review database query times
5. Check for increased traffic patterns

**Scenario B: Error Rate Spike**
```
Alert: "error_rate_high" on order-service
```

Investigation Flow:
1. Get error rate breakdown by status code
2. Identify affected endpoints
3. Search logs for exception patterns
4. Check recent deployments
5. Verify downstream service health

---

## Data Sources Reference

### Prometheus Datasources

| Name | UID | Purpose |
|------|-----|---------|
| UMBQuerier-Luckin | `df8o21agxtkw0d` | Primary infrastructure metrics |
| prometheus_redis | `ff6p0gjt24phce` | Redis cluster metrics |

### Database Servers

| Type | Count | Access |
|------|-------|--------|
| MySQL | 61 servers | mcp-db-gateway |
| PostgreSQL | 3 servers | mcp-db-gateway |
| Redis | 74 clusters | mcp-db-gateway |

### AWS Services

| Service | Access Method | Purpose |
|---------|---------------|---------|
| CloudWatch Logs | cloudwatch-server MCP | Application and system logs |
| CloudWatch Metrics | cloudwatch-server MCP | AWS-native metrics |
| EKS | eks-server MCP | Kubernetes API access |

---

## Troubleshooting Workflows

### Cross-Domain Investigation Flow

```
┌─────────────────┐
│   Alert Fired   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│  Identify Type  │────▶│  Select Skill   │
└────────┬────────┘     └────────┬────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐     ┌─────────────────┐
│ Primary Analysis│────▶│ Cross-Reference │
└────────┬────────┘     └────────┬────────┘
         │                       │
         │    ┌──────────────────┘
         ▼    ▼
┌─────────────────┐
│ Root Cause Found│
│   or Escalate   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Generate Report │
└─────────────────┘
```

### Common Investigation Paths

**Application Slow → Database Issue**
```
/investigate-apm service-x
  └─→ High latency on DB calls detected
      └─→ /investigate-rds service-db
          └─→ Connection exhaustion found
              └─→ Root cause: Connection pool leak
```

**Pod OOM → Host Memory Pressure**
```
/investigate-k8s ns/pod-name
  └─→ OOMKilled, but limits seem adequate
      └─→ /investigate-ec2 node-ip
          └─→ Node memory pressure affecting all pods
              └─→ Root cause: Node needs more memory
```

---

## Best Practices

### 1. Always Validate Alert Context
Don't trust alert names blindly. A "memory alert" might actually be triggered by disk issues.

### 2. Use Parallel Queries
When gathering metrics, request multiple independent queries in a single prompt to speed up investigation.

### 3. Check Siblings First
Often the first sign of systemic issues appears as one instance failing before others.

### 4. Document Evidence
Always capture the specific metric values and timestamps that support your conclusions.

### 5. Follow Escalation Paths
Know when to escalate:
- L0 service impacted > 5 minutes
- Multiple systems affected
- Root cause unclear after initial analysis
- Security incident indicators

### 6. Cross-Reference Data Sources
Use multiple data sources to confirm findings:
- Prometheus metrics + CloudWatch metrics
- Application logs + Database slow logs
- K8s events + Pod logs

---

## Integration Guide

### Installing Skills

Skills are auto-discovered from `~/.claude/skills/`:

```
~/.claude/skills/
├── ec2-alert-investigation/
│   └── SKILL.md
├── rds-alert-investigation/
│   └── SKILL.md
├── k8s-alert-investigation/
│   └── SKILL.md
├── redis-alert-investigation/
│   └── SKILL.md
├── elasticsearch-alert-investigation/
│   └── SKILL.md
└── apm-alert-investigation/
    └── SKILL.md
```

### Installing Commands

Commands go in `~/.claude/commands/`:

```
~/.claude/commands/
├── investigate-ec2.md
├── investigate-rds.md
├── investigate-k8s.md
├── investigate-redis.md
├── investigate-elasticsearch.md
└── investigate-apm.md
```

### Required MCP Servers

Ensure these MCP servers are configured:

1. **Grafana MCP** - For Prometheus queries
2. **mcp-db-gateway** - For MySQL/PostgreSQL/Redis access
3. **cloudwatch-server** - For CloudWatch logs and metrics
4. **eks-server** - For EKS/Kubernetes access

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 6.0 | 2025-01 | Added parallel execution, cross-skill integration |
| 5.0 | 2025-01 | Enhanced error handling, escalation criteria |
| 4.0 | 2024-12 | Added sibling comparison |
| 3.0 | 2024-11 | Initial release |

---

## Contributing

To improve these skills:

1. Fork the repository
2. Update skill files with enhancements
3. Test with real alert scenarios
4. Submit pull request with examples

---

## Support

For issues or questions:
- Create an issue in the repository
- Contact the DevOps team
- Check the #alert-investigation Slack channel
