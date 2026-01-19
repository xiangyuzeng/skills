---
name: k8s-alert-investigation
description: This skill should be used when the user asks to "investigate Kubernetes alert", "debug K8s pods", "check EKS cluster", "analyze pod crashes", mentions Kubernetes/EKS issues, pod failures, OOMKilled, CrashLoopBackOff, pending pods, node problems, or receives alerts about container orchestration including resource limits, scheduling failures, or deployment issues.
version: 3.0.0
---

# Kubernetes/EKS Alert Investigation Skill v3.0

## Overview

Comprehensive protocol for investigating Kubernetes alerts with multi-layer analysis from cluster to container level, leveraging EKS MCP tools and Prometheus metrics.

## Quick Reference

| Data Source | Access Method | Purpose |
|-------------|---------------|---------|
| EKS Cluster | `prod-worker01-eks-us` | K8s API access |
| Prometheus | `df8o21agxtkw0d` (kube-state-metrics) | Cluster metrics |
| CloudWatch | Container Insights | AWS-native metrics |
| DevOps DB | `aws-luckyus-devops-rw` | Service topology |

## Alert Categories

| Category | Trigger Conditions | Priority |
|----------|-------------------|----------|
| OOMKilled | Container memory limit exceeded | L0/L1 |
| CrashLoopBackOff | Repeated restarts > 5 in 10min | L0/L1 |
| Pending Pods | Unscheduled > 5min | L1/L2 |
| Node NotReady | Node unavailable | L0 |
| CPU Throttling | Throttled > 25% of time | L1/L2 |
| Eviction | Pod evicted from node | L1 |

## Investigation Protocol

### Phase 1: Alert Validation (Max 30s)

Extract from alert payload:
- `namespace`, `pod_name`, `container_name`
- `node_name`, `deployment/statefulset`
- Alert type, restart count, last state

Determine priority based on namespace:
- **L0**: `payment`, `order`, `user-auth`
- **L1**: `inventory`, `promotion`, `gateway`
- **L2**: `monitoring`, `logging`, `batch`

### Phase 2: Cluster & Node Health (Parallel)

**Use EKS MCP Tools**:
```
list_k8s_resources(cluster_name="prod-worker01-eks-us", kind="Node", api_version="v1")
```

**Prometheus Node Metrics**:
```promql
# Node conditions (parallel queries)
kube_node_status_condition{condition="Ready",status="true"}
kube_node_status_allocatable{resource="memory"}
kube_node_status_allocatable{resource="cpu"}

# Node resource pressure
sum by(node)(container_memory_working_set_bytes) / sum by(node)(kube_node_status_allocatable{resource="memory"})
```

### Phase 3: Pod Investigation (Parallel)

**Get Pod Details**:
```
manage_k8s_resource(operation="read", cluster_name="prod-worker01-eks-us",
                    kind="Pod", api_version="v1", name="$POD", namespace="$NS")
```

**Get Pod Events**:
```
get_k8s_events(cluster_name="prod-worker01-eks-us", kind="Pod",
               name="$POD", namespace="$NS")
```

**Prometheus Pod Metrics**:
```promql
# Container status
kube_pod_container_status_restarts_total{namespace="$NS",pod="$POD"}
kube_pod_container_status_waiting_reason{namespace="$NS",pod="$POD"}
kube_pod_container_status_terminated_reason{namespace="$NS",pod="$POD"}
```

### Phase 4: Resource Analysis

**Memory Analysis**:
```promql
# Current vs Limit
container_memory_working_set_bytes{namespace="$NS",pod="$POD"}
kube_pod_container_resource_limits{namespace="$NS",pod="$POD",resource="memory"}

# Memory trend (last 1h)
rate(container_memory_working_set_bytes{namespace="$NS",pod="$POD"}[5m])
```

**CPU Analysis**:
```promql
# CPU usage vs request/limit
rate(container_cpu_usage_seconds_total{namespace="$NS",pod="$POD"}[5m])
kube_pod_container_resource_requests{namespace="$NS",pod="$POD",resource="cpu"}
kube_pod_container_resource_limits{namespace="$NS",pod="$POD",resource="cpu"}

# Throttling
rate(container_cpu_cfs_throttled_seconds_total{namespace="$NS",pod="$POD"}[5m])
```

### Phase 5: Application Logs

**Get Container Logs**:
```
get_pod_logs(cluster_name="prod-worker01-eks-us", namespace="$NS",
             pod_name="$POD", container_name="$CONTAINER", tail_lines=100)
```

**Search for Error Patterns**:
- `OutOfMemoryError`, `OOM`, `killed`
- `Exception`, `Error`, `FATAL`
- `Connection refused`, `timeout`
- Stack traces with line numbers

### Phase 6: Workload Configuration

**Get Deployment/StatefulSet**:
```
manage_k8s_resource(operation="read", cluster_name="prod-worker01-eks-us",
                    kind="Deployment", api_version="apps/v1",
                    name="$DEPLOYMENT", namespace="$NS")
```

Check for:
- Resource requests/limits adequacy
- Replica count vs desired
- Rolling update strategy
- Liveness/readiness probe configuration
- Environment variables for dependencies

### Phase 7: Dependency Health Check

```sql
-- Get service dependencies from DevOps DB
SELECT dependency_type, dependency_name, endpoint
FROM service_dependencies
WHERE service_name = '$SERVICE_NAME';
```

Check health of:
- Database endpoints (via RDS investigation if needed)
- Redis clusters (via Redis investigation if needed)
- External APIs (probe endpoints)

## Alert-Specific Playbooks

### OOMKilled
1. Check `container_memory_working_set_bytes` trend
2. Review memory limits vs actual usage patterns
3. Check for memory leaks (continuous growth pattern)
4. **Action**: Increase limits or fix memory leak

### CrashLoopBackOff
1. Check previous container logs (`previous=true`)
2. Look for startup errors or dependency failures
3. Verify config maps and secrets exist
4. **Action**: Fix startup issue or dependencies

### Pending Pods
1. Check `kube_pod_status_phase{phase="Pending"}`
2. Look for scheduling events
3. Verify node resources available
4. Check node selectors/taints/tolerations
5. **Action**: Add nodes or adjust scheduling constraints

### CPU Throttling
1. Check `container_cpu_cfs_throttled_periods_total`
2. Compare CPU limits to actual needs
3. Identify CPU-intensive operations
4. **Action**: Increase CPU limits or optimize code

## Cross-Skill Integration

If investigation reveals:
- **Database connectivity issues** → Suggest `/investigate-rds`
- **Cache timeout errors** → Suggest `/investigate-redis`
- **Host node resource issues** → Suggest `/investigate-ec2`
- **Application performance issues** → Suggest `/investigate-apm`

## Error Handling

| Failure Mode | Fallback Action |
|--------------|----------------|
| EKS API unavailable | Use Prometheus kube-state-metrics only |
| Prometheus gaps | Use CloudWatch Container Insights |
| Logs inaccessible | Check CloudWatch Logs for container output |

## Escalation Criteria

Escalate immediately if:
- All replicas of L0 service are failing
- Node NotReady affecting multiple services
- Cluster-wide scheduling failures
- Persistent volume issues affecting data

## Report Template

```markdown
# Kubernetes Investigation Report

## Alert Summary
- **Cluster**: [EKS Cluster Name]
- **Namespace/Pod**: [ns]/[pod-name]
- **Workload**: [Deployment/StatefulSet name]
- **Alert Type**: [OOMKilled/CrashLoopBackOff/etc]
- **Restart Count**: [N]

## Pod Status
| Container | Status | Restarts | Last State |
|-----------|--------|----------|------------|
| [name] | [Running/Waiting] | [N] | [reason] |

## Resource Utilization
| Resource | Request | Limit | Current | Status |
|----------|---------|-------|---------|--------|
| Memory | XMi | YMi | ZMi | [OK/WARN] |
| CPU | Xm | Ym | Zm | [OK/WARN] |

## Recent Events
1. [Event 1 with timestamp]
2. [Event 2 with timestamp]

## Log Analysis
[Key error patterns found]

## Root Cause
[Determined cause with evidence]

## Recommendations
1. **Immediate**: [Adjust limits / Restart pod]
2. **Short-term**: [Update deployment config]
3. **Long-term**: [Optimize application / Add HPA]
```
