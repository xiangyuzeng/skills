# K8s Alert Investigation Skill

> Version: 2.0 | Edition: Kubernetes (EKS) Diagnosis | Platform: Amazon EKS Clusters

## Skill Trigger

Use this skill when you receive alerts related to Kubernetes/EKS, including:
- **OOMKilled**: Container memory limit exceeded (most common K8s alert)
- **CrashLoopBackOff**: Pod repeatedly failing to start
- **Pending Pods**: Pods unable to be scheduled
- **Node Issues**: Node NotReady, resource pressure, disk pressure
- **Resource Alerts**: CPU throttling, memory pressure, eviction warnings

---

## Data Access Note

**Primary Data Source**: Prometheus (kube-state-metrics)
- EKS direct API access may be restricted by IAM policies
- Kubernetes metrics are available via Prometheus kube-state-metrics
- Container metrics available via cadvisor metrics in Prometheus

**Verified Cluster**: `prod-worker01-eks-us`

**Verified Namespaces** (40+ available):
- Production: `rd-sales`, `rd-finance`, `rd-iot`, `rd-supplychains`, `rd-recommend`
- System: `kube-system`, `amazon-cloudwatch`, `amazon-guardduty`
- Infrastructure: `gatekeeper-system`, `external-secrets`

---

## Investigation Protocol

### Phase 0: Alert Validation

**Objective**: Parse alert and identify Kubernetes resource details

**Actions**:
1. Parse the alert payload to extract:
   - `alertname`: The specific alert type
   - `cluster`: EKS cluster name
   - `namespace`: Kubernetes namespace
   - `pod`: Pod name (if applicable)
   - `deployment`/`statefulset`: Workload name
   - `container`: Container name (for multi-container pods)
   - `node`: Node name (for node-level alerts)
   - `severity`: Alert level
   - `startsAt`: Alert start timestamp

2. Determine investigation path based on alert type:

| Alert Type | Primary Focus |
|------------|---------------|
| OOMKilled | Container memory limits, memory leaks |
| CrashLoopBackOff | Container startup, dependencies, config |
| Pending | Scheduling, resources, node capacity |
| Node NotReady | Node health, kubelet, system resources |
| CPU Throttling | Resource requests/limits, application tuning |

3. Determine service priority:
```sql
-- Server: aws-luckyus-devops-rw
SELECT service_name, namespace, level, owner, oncall_group
FROM k8s_service_registry
WHERE namespace = '<namespace>' AND deployment = '<deployment>';
```

**Priority Classification**:
| Priority | Response Time | Examples |
|----------|---------------|----------|
| L0 | < 15 minutes | Payment services, Core APIs, User Auth |
| L1 | < 30 minutes | Supporting services, Internal APIs |
| L2 | < 2 hours | Background jobs, Dev/Test workloads |

---

### Phase 1: Cluster & Node Health Assessment

**Objective**: Verify cluster and node-level health using Prometheus kube-state-metrics

**Actions**:

1. **Check Node Status via Prometheus** (use mcp__grafana__query_prometheus):
```promql
-- Datasource UID: df8o21agxtkw0d (UMBQuerier-Luckin)
-- List all nodes and their status
kube_node_info

-- Node ready condition (1 = Ready, 0 = NotReady)
kube_node_status_condition{condition="Ready",status="true"}

-- Check for node pressure conditions
kube_node_status_condition{condition="MemoryPressure",status="true"}
kube_node_status_condition{condition="DiskPressure",status="true"}
kube_node_status_condition{condition="PIDPressure",status="true"}
```

2. **Get Node Resource Capacity**:
```promql
-- Node allocatable resources
kube_node_status_allocatable{resource="cpu"}
kube_node_status_allocatable{resource="memory"}
kube_node_status_allocatable{resource="pods"}

-- Node capacity
kube_node_status_capacity{resource="cpu"}
kube_node_status_capacity{resource="memory"}
```

Key node conditions to check:
- `Ready`: Node can accept pods
- `MemoryPressure`: Node is running low on memory
- `DiskPressure`: Node is running low on disk space
- `PIDPressure`: Too many processes on the node
- `NetworkUnavailable`: Node network is not configured correctly

3. **Query Node System Metrics** (use mcp__grafana__query_prometheus):
```promql
-- Datasource UID: df8o21agxtkw0d
-- Node CPU usage (by node label from kube-state-metrics)
100 - (avg by (node) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

-- Node memory usage
100 - ((node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100)

-- Node disk usage
100 - ((node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100)

-- Kubelet status
up{job="kubelet"}
```

4. **Check System Pods via Prometheus**:
```promql
-- List kube-system pods
kube_pod_info{namespace="kube-system"}

-- Pod status in kube-system
kube_pod_status_phase{namespace="kube-system"}
```

**If EKS API is available** (optional, requires IAM permissions):
Use `mcp__eks-server__list_k8s_resources` with cluster_name `prod-worker01-eks-us`

---

### Phase 2: Pod Investigation

**Objective**: Detailed investigation of affected pod(s) using Prometheus kube-state-metrics

**Actions**:

1. **List Pods in Namespace** (use mcp__grafana__query_prometheus):
```promql
-- Datasource UID: df8o21agxtkw0d (UMBQuerier-Luckin)
-- List all pods in namespace with details
kube_pod_info{namespace="<namespace>"}

-- Filter by specific pod name prefix
kube_pod_info{namespace="<namespace>",pod=~"<pod_prefix>.*"}

-- Get pod owner (deployment, statefulset, etc.)
kube_pod_owner{namespace="<namespace>"}
```

2. **Get Pod Status and Phase**:
```promql
-- Pod phase (Pending=1 when pending, Running=1 when running, etc.)
kube_pod_status_phase{namespace="<namespace>",pod=~"<pod_name>.*"}

-- Pod ready condition
kube_pod_status_ready{namespace="<namespace>",pod=~"<pod_name>.*"}

-- Pod scheduled condition
kube_pod_status_scheduled{namespace="<namespace>",pod=~"<pod_name>.*"}
```

Key information to extract from kube-state-metrics:
- Pod phase (Pending, Running, Succeeded, Failed, Unknown)
- Container statuses (waiting, running, terminated)
- Restart count
- Last termination reason
- Node assignment

3. **Get Container Status and Restarts**:
```promql
-- Container restart count (high values indicate CrashLoopBackOff)
kube_pod_container_status_restarts_total{namespace="<namespace>",pod=~"<pod_name>.*"}

-- Container waiting reason (CrashLoopBackOff, ImagePullBackOff, etc.)
kube_pod_container_status_waiting_reason{namespace="<namespace>",pod=~"<pod_name>.*"}

-- Container terminated reason (OOMKilled, Error, Completed)
kube_pod_container_status_terminated_reason{namespace="<namespace>",pod=~"<pod_name>.*"}

-- Container ready status
kube_pod_container_status_ready{namespace="<namespace>",pod=~"<pod_name>.*"}
```

Common termination reasons from kube-state-metrics:
| Metric Value | Indicates |
|--------------|-----------|
| `reason="OOMKilled"` | Container exceeded memory limit |
| `reason="Error"` | Container exited with error |
| `reason="Completed"` | Container completed successfully |
| `reason="CrashLoopBackOff"` | Container keeps crashing |
| `reason="ImagePullBackOff"` | Cannot pull container image |

4. **Check Pod Age and Creation Time**:
```promql
-- Pod start time (Unix timestamp)
kube_pod_start_time{namespace="<namespace>",pod=~"<pod_name>.*"}

-- Calculate pod age
time() - kube_pod_start_time{namespace="<namespace>",pod=~"<pod_name>.*"}
```

**If EKS API is available** (optional):
Use `mcp__eks-server__get_k8s_events` for detailed event messages

---

### Phase 3: Resource Analysis

**Objective**: Analyze CPU and memory usage vs requests/limits

**Actions**:

1. **Get Container Resource Metrics** (use mcp__grafana__query_prometheus):
```promql
-- Container CPU usage
sum(rate(container_cpu_usage_seconds_total{namespace="<namespace>",pod="<pod_name>"}[5m])) by (container)

-- Container memory usage
sum(container_memory_working_set_bytes{namespace="<namespace>",pod="<pod_name>"}) by (container)

-- Container memory limit
sum(kube_pod_container_resource_limits{namespace="<namespace>",pod="<pod_name>",resource="memory"}) by (container)

-- Container memory request
sum(kube_pod_container_resource_requests{namespace="<namespace>",pod="<pod_name>",resource="memory"}) by (container)

-- CPU throttling
sum(rate(container_cpu_cfs_throttled_seconds_total{namespace="<namespace>",pod="<pod_name>"}[5m])) by (container)
```

2. **Compare Usage vs Limits**:

Calculate ratios:
- Memory Usage / Memory Limit > 90%: High risk of OOMKill
- CPU Usage / CPU Limit consistently at 100%: Being throttled
- Memory Usage / Memory Request > 100%: Overcommitted

3. **Historical Resource Trend** (last 24 hours):
```promql
-- Memory usage trend
avg_over_time(container_memory_working_set_bytes{namespace="<namespace>",pod=~"<pod_prefix>.*"}[24h])

-- CPU usage trend
avg_over_time(rate(container_cpu_usage_seconds_total{namespace="<namespace>",pod=~"<pod_prefix>.*"}[5m])[24h:5m])
```

---

### Phase 4: Workload Configuration Review

**Objective**: Review Deployment/StatefulSet configuration using Prometheus

**Actions**:

1. **Get Deployment Replica Status** (use mcp__grafana__query_prometheus):
```promql
-- Datasource UID: df8o21agxtkw0d
-- Deployment desired replicas
kube_deployment_spec_replicas{namespace="<namespace>",deployment="<deployment_name>"}

-- Deployment available replicas
kube_deployment_status_replicas_available{namespace="<namespace>",deployment="<deployment_name>"}

-- Deployment unavailable replicas
kube_deployment_status_replicas_unavailable{namespace="<namespace>",deployment="<deployment_name>"}

-- Deployment ready replicas
kube_deployment_status_replicas_ready{namespace="<namespace>",deployment="<deployment_name>"}
```

2. **Get StatefulSet Status** (if applicable):
```promql
-- StatefulSet desired replicas
kube_statefulset_replicas{namespace="<namespace>",statefulset="<statefulset_name>"}

-- StatefulSet ready replicas
kube_statefulset_status_replicas_ready{namespace="<namespace>",statefulset="<statefulset_name>"}

-- StatefulSet current replicas
kube_statefulset_status_replicas_current{namespace="<namespace>",statefulset="<statefulset_name>"}
```

3. **Check Replica Health Mismatch**:
```promql
-- Find deployments where desired != available (indicates problems)
kube_deployment_spec_replicas{namespace="<namespace>"}
  - kube_deployment_status_replicas_available{namespace="<namespace>"} > 0
```

4. **Review Resource Requests and Limits**:
```promql
-- Container resource requests
kube_pod_container_resource_requests{namespace="<namespace>",pod=~"<pod_prefix>.*"}

-- Container resource limits
kube_pod_container_resource_limits{namespace="<namespace>",pod=~"<pod_prefix>.*"}
```

5. **Check Resource Quotas** (via Prometheus):
```promql
-- Namespace resource quota used
kube_resourcequota{namespace="<namespace>",type="used"}

-- Namespace resource quota hard limit
kube_resourcequota{namespace="<namespace>",type="hard"}
```

6. **Check HPA (Horizontal Pod Autoscaler) Status**:
```promql
-- HPA current replicas
kube_horizontalpodautoscaler_status_current_replicas{namespace="<namespace>"}

-- HPA desired replicas
kube_horizontalpodautoscaler_status_desired_replicas{namespace="<namespace>"}

-- HPA max replicas (check if maxed out)
kube_horizontalpodautoscaler_spec_max_replicas{namespace="<namespace>"}

-- HPA min replicas
kube_horizontalpodautoscaler_spec_min_replicas{namespace="<namespace>"}
```

---

### Phase 5: Application Logs Analysis

**Objective**: Analyze container logs for errors and patterns

**Actions**:

1. **Get Pod Logs** (use mcp__eks-server__get_pod_logs):
```
cluster_name: <cluster>
namespace: <namespace>
pod_name: <pod_name>
container_name: <container>  # if multi-container
tail_lines: 200
previous: false  # set true for crashed container logs
```

2. **Get Previous Container Logs** (for crashed containers):
```
cluster_name: <cluster>
namespace: <namespace>
pod_name: <pod_name>
previous: true
tail_lines: 500
```

3. **Search for Error Patterns**:
Common patterns to look for:
- `OutOfMemoryError` / `OOM` (Java)
- `FATAL` / `PANIC`
- `Connection refused` / `Connection timeout`
- `Killed` / `SIGKILL` / `SIGTERM`
- Stack traces and exceptions
- Startup failures

4. **Query CloudWatch Logs** (if logs are shipped to CloudWatch):
```
-- Use mcp__cloudwatch-server__execute_log_insights_query
-- Log group: /aws/containerinsights/<cluster>/application

fields @timestamp, @message, kubernetes.pod_name
| filter kubernetes.namespace_name = "<namespace>"
| filter kubernetes.pod_name like /<pod_prefix>/
| filter @message like /error|Error|ERROR|exception|Exception|fatal|Fatal/
| sort @timestamp desc
| limit 100
```

---

### Phase 6: Dependency Analysis

**Objective**: Check service dependencies (databases, caches, external services)

**Actions**:

1. **Check Service Endpoints**:
```
cluster_name: <cluster>
kind: Endpoints
api_version: v1
namespace: <namespace>
name: <service_name>
```

2. **Query Database Dependencies** (if applicable):
```sql
-- Server: aws-luckyus-devops-rw
SELECT db_host, db_name, connection_pool_size
FROM k8s_service_db_mapping
WHERE namespace = '<namespace>' AND deployment = '<deployment>';
```

3. **Check database connectivity** from pod logs:
- Connection pool exhaustion
- Database timeouts
- Authentication failures

4. **Check Redis Dependencies**:
```
-- Use mcp__mcp-db-gateway__redis_command on related cluster
PING
INFO clients
INFO memory
```

5. **Check External Service Health**:
```promql
-- HTTP client metrics (if available)
rate(http_client_requests_seconds_count{namespace="<namespace>",pod=~"<pod_prefix>.*",status=~"5.."}[5m])

-- Outbound connection errors
rate(tcp_connection_errors_total{namespace="<namespace>",pod=~"<pod_prefix>.*"}[5m])
```

---

### Phase 7: Alert Correlation

**Objective**: Find related alerts from the same time window

**Actions**:

1. **List Related Alerts** from Grafana:
```
-- Use mcp__grafana__list_alert_rules with label selectors
label_selectors: [{"filters": [{"name": "namespace", "type": "=", "value": "<namespace>"}]}]
```

2. **Query Alert Log**:
```sql
-- Server: aws-luckyus-devops-rw
SELECT alert_name, labels, severity, starts_at, ends_at
FROM alert_log
WHERE starts_at >= NOW() - INTERVAL 2 HOUR
  AND (
    labels LIKE '%<namespace>%'
    OR labels LIKE '%<cluster>%'
    OR labels LIKE '%<deployment>%'
  )
ORDER BY starts_at DESC;
```

3. **Check for Cascade Patterns**:
- Node alerts before pod alerts?
- Multiple pods in same namespace affected?
- Deployment-wide vs single pod issue?

---

### Phase 8: Workload-Specific Playbooks

**Objective**: Execute alert-type-specific investigation procedures

#### Playbook 8a: OOMKilled Investigation

**Most Common K8s Issue - Execute thoroughly**

1. **Confirm OOMKilled**:
```
-- Check pod events for OOMKilled
-- Check container lastState.terminated.reason = "OOMKilled"
-- Exit code 137 = SIGKILL (OOM)
```

2. **Analyze Memory Usage Pattern**:
```promql
-- Memory usage over time (1 hour before OOM)
container_memory_working_set_bytes{namespace="<namespace>",pod=~"<pod_prefix>.*"}

-- Memory vs Limit
container_memory_working_set_bytes{namespace="<namespace>",pod=~"<pod_prefix>.*"} /
kube_pod_container_resource_limits{namespace="<namespace>",pod=~"<pod_prefix>.*",resource="memory"}
```

3. **Check for Memory Leaks**:
- Continuously increasing memory over restarts
- No plateau in memory growth
- Fast memory growth after startup

4. **Recommendations**:
- Increase memory limit if usage is legitimate
- Investigate memory leaks if growth is abnormal
- Review heap settings for JVM applications
- Check for connection/resource leaks

#### Playbook 8b: CrashLoopBackOff Investigation

1. **Check Restart Count and Pattern**:
- How frequently is it restarting?
- Is there a pattern (immediate crash vs after some time)?

2. **Get Previous Container Logs**:
```
previous: true
tail_lines: 500
```

3. **Check Exit Codes**:
| Exit Code | Meaning |
|-----------|---------|
| 0 | Success (unexpected for running pod) |
| 1 | Application error |
| 137 | SIGKILL (OOMKilled or killed) |
| 143 | SIGTERM (graceful shutdown) |
| 255 | Exit status out of range |

4. **Common Causes**:
- Missing config/secrets
- Database connectivity issues
- Permission problems
- Missing dependencies
- Application bugs

#### Playbook 8c: Pending Pod Investigation

1. **Check Pod Events**:
- `FailedScheduling`: No suitable node
- `Insufficient cpu/memory`: Node capacity issue
- `NodeSelectorMismatch`: No node matches selectors
- `PodTolerationsInsufficent`: Taint/toleration mismatch

2. **Check Node Capacity**:
```promql
-- Allocatable vs used CPU
sum(kube_node_status_allocatable{resource="cpu"}) - sum(kube_pod_container_resource_requests{resource="cpu"})

-- Allocatable vs used memory
sum(kube_node_status_allocatable{resource="memory"}) - sum(kube_pod_container_resource_requests{resource="memory"})
```

3. **Check Node Selectors and Affinity**:
- Review pod nodeSelector
- Review node affinity/anti-affinity rules
- Check if target nodes exist and are Ready

#### Playbook 8d: Node NotReady Investigation

1. **Check Node Conditions**:
All conditions should be `False` except `Ready` which should be `True`:
- MemoryPressure
- DiskPressure
- PIDPressure
- NetworkUnavailable

2. **Check Kubelet**:
```promql
up{job="kubelet",node="<node_name>"}
kubelet_running_pod_count{node="<node_name>"}
```

3. **Check Node System Resources**:
```promql
-- Same as Phase 1 node metrics
```

4. **Check for Recent Events**:
```
-- Node events via get_k8s_events
kind: Node
name: <node_name>
```

---

### Phase 9: Output Report Generation

**Objective**: Generate comprehensive K8s investigation report

**Report Template**:

```markdown
## K8s Investigation Report

### Alert Summary
| Field | Value |
|-------|-------|
| Alert Name | [alertname] |
| Cluster | [cluster_name] |
| Namespace | [namespace] |
| Workload | [deployment/statefulset] |
| Pod | [pod_name] |
| Node | [node_name] |
| Severity | [L0/L1/L2] |
| Start Time | [timestamp] |
| Duration | [duration] |

### Service Context
- **Service Level**: [L0/L1/L2]
- **Service Owner**: [owner]
- **On-Call Group**: [oncall_group]

### Cluster Health
| Metric | Value | Status |
|--------|-------|--------|
| Total Nodes | [X] | - |
| Ready Nodes | [X] | [OK/WARNING] |
| Node Conditions | [list] | [OK/WARNING] |

### Affected Node Analysis
| Metric | Value | Status |
|--------|-------|--------|
| CPU Usage | [X%] | [OK/WARNING/CRITICAL] |
| Memory Usage | [X%] | [OK/WARNING/CRITICAL] |
| Disk Usage | [X%] | [OK/WARNING/CRITICAL] |
| Pod Count | [X/Y] | [OK/WARNING] |

### Pod Analysis

#### Current Status
| Field | Value |
|-------|-------|
| Phase | [Running/Pending/Failed] |
| Ready | [True/False] |
| Restart Count | [X] |
| Age | [duration] |

#### Container Status
| Container | State | Restarts | Last Termination |
|-----------|-------|----------|------------------|
| [name] | [running/waiting/terminated] | [X] | [reason: OOMKilled/Error/None] |

#### Resource Usage
| Resource | Request | Limit | Actual | % of Limit |
|----------|---------|-------|--------|------------|
| CPU | [X] | [Y] | [Z] | [W%] |
| Memory | [X Mi] | [Y Mi] | [Z Mi] | [W%] |

### Event Timeline
| Time | Type | Reason | Message |
|------|------|--------|---------|
| [timestamp] | [Normal/Warning] | [reason] | [message] |

### Workload Configuration
| Field | Value | Assessment |
|-------|-------|------------|
| Replicas | [desired/ready/available] | [OK/DEGRADED] |
| Strategy | [RollingUpdate/Recreate] | - |
| Resource Quota | [used/limit] | [OK/EXCEEDED] |

### Log Analysis

#### Error Patterns Found
| Pattern | Count | Sample |
|---------|-------|--------|
| [pattern] | [X] | [sample message] |

#### Last Lines Before Crash
```
[relevant log lines]
```

### Dependencies Status
| Dependency | Type | Status | Details |
|------------|------|--------|---------|
| [service] | Database | [Healthy/Issues] | [connection count, latency] |
| [cluster] | Redis | [Healthy/Issues] | [memory, connections] |

### Related Alerts
| Alert | Resource | Time | Status |
|-------|----------|------|--------|
| [name] | [pod/node] | [time] | [Firing/Resolved] |

### Root Cause Analysis
**Primary Cause**: [Identified root cause]
**Alert Type Specific**:
- For OOMKilled: [Memory leak / Insufficient limit / Spike in traffic]
- For CrashLoop: [Config error / Dependency failure / Application bug]
- For Pending: [Insufficient resources / Node selector mismatch]
- For Node NotReady: [Node failure / Resource pressure / Network issues]

**Contributing Factors**: [Additional factors]

### Recommendations
| Priority | Action | Owner | Timeline |
|----------|--------|-------|----------|
| P0 | [Immediate action] | [team] | Immediate |
| P1 | [Short-term fix] | [team] | 24 hours |
| P2 | [Long-term solution] | [team] | 1 week |

#### Specific Recommendations by Alert Type

**For OOMKilled**:
- [ ] Increase memory limit to [recommended value]
- [ ] Investigate memory leak in [component]
- [ ] Review JVM heap settings
- [ ] Implement memory profiling

**For CrashLoopBackOff**:
- [ ] Fix configuration issue [detail]
- [ ] Resolve dependency [service]
- [ ] Update application [fix]

**For Pending**:
- [ ] Add node capacity
- [ ] Review resource requests
- [ ] Update node selectors

### Evidence

<details>
<summary>Pod Description</summary>

```yaml
[Pod YAML or key fields]
```

</details>

<details>
<summary>Pod Events</summary>

[Events list]

</details>

<details>
<summary>Container Logs</summary>

```
[Relevant log excerpts]
```

</details>

<details>
<summary>Prometheus Metrics</summary>

[Query results]

</details>

---
*Investigation completed at [timestamp]*
*Skill Version: K8s Alert Investigation SOP v1.0*
```

---

## MCP Tools Reference

| Tool | Server | Purpose | Priority |
|------|--------|---------|----------|
| `mcp__grafana__query_prometheus` | grafana | **PRIMARY**: K8s metrics via kube-state-metrics | High |
| `mcp__cloudwatch-server__execute_log_insights_query` | cloudwatch-server | CloudWatch container logs | High |
| `mcp__mcp-db-gateway__mysql_query` | mcp-db-gateway | Service topology lookup | Medium |
| `mcp__mcp-db-gateway__redis_command` | mcp-db-gateway | Dependency checks (74 clusters) | Medium |
| `mcp__eks-server__list_k8s_resources` | eks-server | List resources (requires IAM) | Optional |
| `mcp__eks-server__get_k8s_events` | eks-server | Get events (requires IAM) | Optional |
| `mcp__eks-server__get_pod_logs` | eks-server | Get pod logs (requires IAM) | Optional |

## Key Prometheus Queries for K8s

```promql
# Datasource UID: df8o21agxtkw0d (UMBQuerier-Luckin)

# List all pods in a namespace
kube_pod_info{namespace="rd-sales"}

# Check pod restart counts (high = CrashLoopBackOff)
kube_pod_container_status_restarts_total{namespace="<namespace>"}

# Check for OOMKilled containers
kube_pod_container_status_terminated_reason{reason="OOMKilled"}

# Check deployment replica status
kube_deployment_status_replicas_unavailable{namespace="<namespace>"} > 0

# Check node conditions
kube_node_status_condition{condition="Ready",status="true"}

# Container memory usage
container_memory_working_set_bytes{namespace="<namespace>"}

# Container CPU usage
rate(container_cpu_usage_seconds_total{namespace="<namespace>"}[5m])
```

## Key EKS Operations (If IAM Permits)

```python
# List pods in namespace (requires eks:DescribeCluster)
mcp__eks-server__list_k8s_resources(
    cluster_name="prod-worker01-eks-us",
    kind="Pod",
    api_version="v1",
    namespace="<namespace>"
)

# Get pod events
mcp__eks-server__get_k8s_events(
    cluster_name="prod-worker01-eks-us",
    kind="Pod",
    name="<pod_name>",
    namespace="<namespace>"
)

# Get pod logs
mcp__eks-server__get_pod_logs(
    cluster_name="prod-worker01-eks-us",
    namespace="<namespace>",
    pod_name="<pod_name>",
    tail_lines=200
)
```

---

## Example Usage

When investigating a K8s alert, invoke this skill with the alert payload:

```
/investigate-k8s {
  "alertname": "KubePodCrashLooping",
  "cluster": "eks-luckin-prod",
  "namespace": "order-service",
  "pod": "order-api-7f8d9c6b4-x2k9m",
  "deployment": "order-api",
  "severity": "critical"
}
```

The skill will execute all 9 phases automatically and generate a comprehensive investigation report.

---

## OOMKilled Quick Reference

Since OOMKilled is the most common K8s alert, here's a quick checklist:

1. **Confirm OOMKilled**: Event shows "OOMKilled", exit code 137
2. **Check memory usage trend**: Was it gradual increase or sudden spike?
3. **Compare to limit**: How close was usage to limit before kill?
4. **Check sibling pods**: Are other replicas also affected?
5. **Review application logs**: Any memory-related errors?
6. **Check for leaks**: Continuous growth across restarts?
7. **Recommend action**:
   - If limit too low: Increase limit
   - If leak detected: Flag for development team
   - If traffic spike: Consider HPA configuration
