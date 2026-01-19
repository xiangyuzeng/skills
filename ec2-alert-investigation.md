---
name: ec2-alert-investigation
description: This skill should be used when the user asks to "investigate EC2 alert", "debug EC2 instance", "check EC2 performance", "analyze EC2 CPU/memory/disk", mentions EC2 instance issues, VM performance problems, or receives alerts about AWS EC2 instances including CPU spikes, memory pressure, disk full, I/O bottlenecks, or network issues.
version: 6.0.0
---

# EC2 Alert Investigation Skill v6.0

## Overview

Comprehensive protocol for investigating EC2 instance alerts with parallel data collection, cross-system correlation, and automated root cause analysis.

## Quick Reference

| Data Source | UID/Server | Purpose |
|-------------|------------|---------|
| Prometheus | `df8o21agxtkw0d` (UMBQuerier-Luckin) | Primary metrics |
| DevOps DB | `aws-luckyus-devops-rw` | Service topology |
| CloudWatch | Logs & Metrics | AWS-native data |
| Redis Cache | `ff6p0gjt24phce` | Cache dependency |

## Alert Categories

| Category | Trigger Conditions | Priority |
|----------|-------------------|----------|
| CPU | > 80% sustained 5min | L1/L2 |
| Memory | > 90% used OR OOM events | L0/L1 |
| Disk | > 90% OR < 5GB free | L0/L1 |
| I/O | await > 100ms OR util > 90% | L1/L2 |
| Network | packet drops > 20/s OR errors | L1/L2 |

## Investigation Protocol

### Phase 1: Alert Validation (Max 30s)

Extract from alert payload:
- `instance_id`, `instance_ip`, `service_name`
- Alert type, threshold, current value
- Timestamp and duration

Determine priority:
- **L0**: Payment, Order, User-Auth services (5min response)
- **L1**: Core business services (15min response)
- **L2**: Supporting services (30min response)

### Phase 2: Parallel Data Collection

**CRITICAL**: Execute these queries in parallel using multiple tool calls in a single message:

```promql
# Query Set A: Resource Utilization (parallel)
node_filesystem_avail_bytes{instance=~"$IP.*"} / node_filesystem_size_bytes{instance=~"$IP.*"} * 100
node_memory_MemAvailable_bytes{instance=~"$IP.*"} / node_memory_MemTotal_bytes{instance=~"$IP.*"} * 100
100 - (avg by(instance)(irate(node_cpu_seconds_total{mode="idle",instance=~"$IP.*"}[5m])) * 100)
```

```promql
# Query Set B: I/O and Network (parallel)
rate(node_disk_io_time_seconds_total{instance=~"$IP.*"}[5m]) * 100
rate(node_network_receive_bytes_total{instance=~"$IP.*"}[5m])
rate(node_network_transmit_bytes_total{instance=~"$IP.*"}[5m])
```

### Phase 3: Sibling Comparison

Compare against peer instances in same service group:

```sql
-- Get sibling instances
SELECT instance_ip, instance_id
FROM ec2_instances
WHERE service_name = '$SERVICE' AND status = 'running'
```

Flag anomalies if affected instance deviates > 1.5 standard deviations from group mean.

### Phase 4: Service-Specific Metrics

**For Java Services**:
```promql
jvm_memory_used_bytes{instance=~"$IP.*"} / jvm_memory_max_bytes{instance=~"$IP.*"}
rate(jvm_gc_pause_seconds_sum{instance=~"$IP.*"}[5m])
```

**For All Services**:
```promql
process_open_fds{instance=~"$IP.*"}
process_max_fds{instance=~"$IP.*"}
```

### Phase 5: Dependency Analysis

```sql
-- Get database dependencies
SELECT db.host, db.port, db.database_name
FROM service_db_mapping sdm
JOIN databases db ON sdm.db_id = db.id
WHERE sdm.service_name = '$SERVICE'
```

```sql
-- Get cache dependencies
SELECT rc.cluster_name, rc.endpoint
FROM service_redis_mapping srm
JOIN redis_clusters rc ON srm.redis_id = rc.id
WHERE srm.service_name = '$SERVICE'
```

### Phase 6: CloudWatch Integration

Cross-reference with AWS metrics:
- `CPUUtilization`, `NetworkIn/Out`, `DiskReadOps/WriteOps`
- Check for EC2 status check failures
- Query system logs for kernel errors

### Phase 7: Root Cause Determination

| Symptom Pattern | Likely Cause | Recommended Action |
|-----------------|--------------|-------------------|
| High CPU + Low Memory | CPU-bound workload | Scale out or optimize code |
| High Memory + Swap | Memory leak | Heap dump analysis |
| High I/O Wait + Disk Full | Storage exhaustion | Expand volume or cleanup |
| Network Drops + Retransmits | Network saturation | Check security groups/NACLs |

## Cross-Skill Integration

If investigation reveals:
- **Database issues** → Suggest `/investigate-rds`
- **Cache issues** → Suggest `/investigate-redis`
- **Kubernetes pod issues** → Suggest `/investigate-k8s`
- **Application errors** → Suggest `/investigate-apm`

## Error Handling

| Failure Mode | Fallback Action |
|--------------|----------------|
| Prometheus unavailable | Use CloudWatch metrics exclusively |
| DevOps DB timeout | Use cached topology from last 24h |
| Missing node exporter | Check if instance is new (<1h old) |

## Escalation Criteria

Escalate immediately if:
- L0 service impacted > 5 minutes
- Multiple instances affected simultaneously
- Root cause unclear after Phase 5
- Security incident indicators present

## Report Template

```markdown
# EC2 Investigation Report

## Alert Summary
- **Instance**: [ID] ([IP])
- **Service**: [Name] (Priority: [L0/L1/L2])
- **Alert Type**: [Category]
- **Duration**: [Time since trigger]

## Key Findings
1. [Primary finding with evidence]
2. [Secondary finding]

## Metrics Snapshot
| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| CPU | X% | 80% | [OK/WARN/CRIT] |
| Memory | X% | 90% | [OK/WARN/CRIT] |
| Disk | X% | 90% | [OK/WARN/CRIT] |

## Root Cause
[Determined cause with supporting evidence]

## Recommendations
1. **Immediate**: [Action]
2. **Short-term**: [Action]
3. **Long-term**: [Action]

## Related Alerts
- [List any correlated alerts]
```
