# Real-World Use Cases

> Practical examples of using alert investigation skills for common scenarios

---

## EC2 Use Cases

### Use Case 1: Disk Space Critical on Order Service

**Alert Received:**
```json
{
  "alertname": "DiskUsedPercent",
  "instance": "10.1.5.42:9100",
  "severity": "critical",
  "service_name": "order-service",
  "job": "order-nodes",
  "mountpoint": "/data",
  "current_value": "94.2"
}
```

**Invocation:**
```
/investigate-ec2 {"alertname": "DiskUsedPercent", "instance": "10.1.5.42:9100", "severity": "critical", "service_name": "order-service", "job": "order-nodes"}
```

**What the Skill Does:**
1. Validates this is an EC2 alert, identifies L0 priority (order-service)
2. Queries Prometheus for all filesystem metrics on this instance
3. Checks inode usage (sometimes the real culprit)
4. Compares with sibling order-service instances to detect anomaly
5. Checks for large log files, old temp files
6. Correlates with any recent deployments or traffic spikes
7. Generates report with root cause and cleanup recommendations

**Expected Findings:**
- Log rotation not configured for `/data/logs/app.log`
- Temp files not cleaned in `/data/tmp`
- Sibling instances at 65% disk (this instance is anomalous)

---

### Use Case 2: Memory Exhaustion with OOM Events

**Alert Received:**
```json
{
  "alertname": "MemUsedPercent",
  "instance": "10.2.3.15:9100",
  "severity": "critical",
  "service_name": "payment-gateway",
  "job": "payment-nodes",
  "current_value": "97.8"
}
```

**What the Skill Does:**
1. Identifies L0 priority (payment-gateway - critical service)
2. Queries memory breakdown: MemTotal, MemAvailable, Buffers, Cached
3. Checks for memory leak patterns (steady increase over time)
4. For Java apps: queries JVM heap usage, GC activity
5. Checks swap usage and OOM killer activity
6. Correlates with traffic patterns
7. Compares with sibling payment-gateway instances

**Expected Findings:**
- JVM heap at 95% of max (memory leak suspected)
- GC overhead increased 3x in last 2 hours
- Recommendation: Restart service, investigate memory leak

---

### Use Case 3: High I/O Wait Causing Application Slowness

**Alert Received:**
```json
{
  "alertname": "IOWaitHigh",
  "instance": "10.3.4.20:9100",
  "severity": "warning",
  "service_name": "analytics-worker",
  "job": "analytics-nodes",
  "current_value": "35.2"
}
```

**What the Skill Does:**
1. Queries disk I/O metrics: read/write IOPS, throughput, await
2. Identifies which disk/mount is causing the I/O bottleneck
3. Checks if this correlates with a batch job or data import
4. Queries database dependencies (analytics-worker likely queries RDS)
5. Triggers RDS skill if database is the source of I/O pressure

---

## RDS Use Cases

### Use Case 4: Database Connection Pool Exhausted

**Alert Received:**
```json
{
  "alertname": "RDSConnectionHigh",
  "instance": "aws-luckyus-order-rw",
  "severity": "critical",
  "current_connections": 485,
  "max_connections": 500
}
```

**Invocation:**
```
/investigate-rds {"alertname": "RDSConnectionHigh", "instance": "aws-luckyus-order-rw", "severity": "critical", "current_connections": 485, "max_connections": 500}
```

**What the Skill Does:**
1. Validates RDS alert, queries connection metrics from Prometheus
2. Runs `SHOW PROCESSLIST` to identify connection sources
3. Groups connections by user, host, database
4. Identifies if specific application is leaking connections
5. Checks for long-running queries holding connections
6. Queries connection trend over past 24 hours
7. Identifies dependent services from DevOps database

**Expected Findings:**
- 340 connections from `order-service` (normally 150)
- 50 connections in `Sleep` state > 300 seconds
- Connection pool leak after recent deployment
- Recommendation: Restart order-service pods, fix connection leak

---

### Use Case 5: Slow Query Performance Degradation

**Alert Received:**
```json
{
  "alertname": "SlowQueryRateHigh",
  "instance": "aws-luckyus-inventory-rw",
  "severity": "warning",
  "slow_queries_per_min": 45,
  "baseline": 5
}
```

**What the Skill Does:**
1. Queries CloudWatch slow query logs for this RDS instance
2. Analyzes slow query patterns (which queries, frequency)
3. Identifies missing indexes using `EXPLAIN`
4. Checks table statistics freshness
5. Correlates with recent schema changes or traffic spikes
6. Queries InnoDB buffer pool hit ratio
7. Checks if replicas are available to offload read traffic

**Expected Findings:**
```sql
-- Top slow query identified
SELECT * FROM inventory WHERE product_id IN (...)
-- Missing index on product_id + warehouse_id compound key
-- Recommendation: CREATE INDEX idx_product_warehouse ON inventory(product_id, warehouse_id)
```

---

### Use Case 6: Replication Lag Alert

**Alert Received:**
```json
{
  "alertname": "RDSReplicaLag",
  "instance": "aws-luckyus-user-ro-1",
  "severity": "warning",
  "lag_seconds": 120,
  "master": "aws-luckyus-user-rw"
}
```

**What the Skill Does:**
1. Queries replication status from Prometheus
2. Runs `SHOW SLAVE STATUS` on replica
3. Identifies if lag is from write volume or replica capacity
4. Checks master binary log position vs replica position
5. Correlates with any bulk operations on master
6. Checks replica instance specs vs master
7. Identifies services reading from this replica

**Expected Findings:**
- Master processing 5x normal writes (bulk import running)
- Replica CPU at 95% (single-threaded replication bottleneck)
- Recommendation: Wait for bulk import to complete, consider parallel replication

---

## K8s (EKS) Use Cases

### Use Case 7: Pod OOMKilled Repeatedly

**Alert Received:**
```json
{
  "alertname": "KubePodOOMKilled",
  "namespace": "production",
  "pod": "recommendation-service-7f9d8c6b5-x2k4m",
  "container": "recommendation-api",
  "severity": "critical",
  "restart_count": 5
}
```

**Invocation:**
```
/investigate-k8s {"alertname": "KubePodOOMKilled", "namespace": "production", "pod": "recommendation-service-7f9d8c6b5-x2k4m", "container": "recommendation-api", "severity": "critical"}
```

**What the Skill Does:**
1. Queries pod status and container state from EKS API
2. Gets Kubernetes events showing OOMKilled reason
3. Queries Container Insights for memory usage pattern
4. Compares memory requests/limits with actual usage
5. Gets container logs before OOM for context
6. Checks if HPA is at max replicas
7. Identifies if specific requests trigger OOM

**Expected Findings:**
- Memory limit: 512Mi, actual peak usage: 520Mi
- Memory leak after processing large recommendation batches
- Recommendation: Increase memory limit to 1Gi, investigate leak

---

### Use Case 8: Pod Stuck in Pending State

**Alert Received:**
```json
{
  "alertname": "KubePodPending",
  "namespace": "production",
  "pod": "search-service-5c8f7d6b4-abc12",
  "severity": "warning",
  "pending_duration": "15m"
}
```

**What the Skill Does:**
1. Queries pod status and conditions
2. Gets pod events showing scheduling failures
3. Checks node resources (CPU, memory available)
4. Analyzes pod resource requests vs cluster capacity
5. Checks for node taints and pod tolerations
6. Checks node affinity rules
7. Identifies resource quotas in namespace

**Expected Findings:**
- Event: `FailedScheduling: Insufficient cpu`
- Pod requests 2 CPU, no nodes have 2 CPU available
- 3 nodes fully utilized, HPA maxed out
- Recommendation: Scale cluster or reduce CPU request

---

### Use Case 9: CrashLoopBackOff for New Deployment

**Alert Received:**
```json
{
  "alertname": "KubePodCrashLooping",
  "namespace": "staging",
  "pod": "checkout-service-6d9f8e7c4-def45",
  "container": "checkout-api",
  "severity": "critical",
  "restart_count": 8
}
```

**What the Skill Does:**
1. Gets pod status showing `CrashLoopBackOff`
2. Fetches container logs (current and previous)
3. Gets pod events for error messages
4. Checks recent deployment/rollout history
5. Compares with previous healthy pod configuration
6. Checks ConfigMaps and Secrets for changes
7. Verifies database/service dependencies are accessible

**Expected Findings:**
```
Container Logs:
Error: ECONNREFUSED connecting to redis://redis-cluster:6379
Config: REDIS_HOST=redis-cluster (was redis-master)

Root Cause: ConfigMap changed Redis host to non-existent service
Recommendation: Rollback ConfigMap or fix Redis service name
```

---

### Use Case 10: Node NotReady Affecting Multiple Pods

**Alert Received:**
```json
{
  "alertname": "KubeNodeNotReady",
  "node": "ip-10-0-1-123.ec2.internal",
  "severity": "critical",
  "condition": "Ready=False",
  "affected_pods": 12
}
```

**What the Skill Does:**
1. Queries node conditions (Ready, MemoryPressure, DiskPressure, PIDPressure)
2. Gets node events for kubelet errors
3. Checks node resource usage from Container Insights
4. Lists all pods on the affected node
5. Checks if pods are being evicted/rescheduled
6. Correlates with EC2 instance health
7. Triggers EC2 skill for underlying instance investigation

**Expected Findings:**
- Node condition: `DiskPressure=True`
- Kubelet evicting pods due to disk threshold
- Underlying EC2 instance disk at 95%
- Cross-reference with EC2 skill for root cause

---

## Cross-Skill Investigation Examples

### Example A: Application Slow - Multi-Layer Investigation

**Initial Symptom:** Order service response time increased 5x

**Investigation Flow:**
```
1. Start with EC2 Skill (application runs on EC2)
   → CPU normal, Memory normal, Disk normal
   → I/O Wait elevated (15%)

2. EC2 Skill identifies database dependency
   → Triggers RDS Skill for aws-luckyus-order-rw

3. RDS Skill finds:
   → Slow query rate 10x baseline
   → Missing index on orders.customer_id

4. Root Cause: Missing database index
5. Action: Add index, monitor improvement
```

### Example B: Microservice Cascading Failure

**Initial Symptom:** Multiple K8s pods restarting

**Investigation Flow:**
```
1. Start with K8s Skill (pods in EKS)
   → Multiple pods OOMKilled
   → Memory usage spikes correlate with request timeouts

2. K8s Skill identifies upstream dependency timeout
   → Requests backing up, memory growing

3. Trace to RDS database
   → Triggers RDS Skill for aws-luckyus-catalog-rw

4. RDS Skill finds:
   → Database connections maxed out
   → Long-running query blocking others

5. Root Cause: Blocking query causing cascading timeouts
6. Action: Kill blocking query, add query timeout
```

---

## Summary Table

| Use Case | Primary Skill | Secondary Skill | Key Finding |
|----------|---------------|-----------------|-------------|
| Disk Full | EC2 | - | Log rotation missing |
| Memory Exhaustion | EC2 | - | JVM memory leak |
| I/O Wait High | EC2 | RDS | Database query causing I/O |
| Connection Pool | RDS | - | Connection leak in app |
| Slow Queries | RDS | - | Missing index |
| Replication Lag | RDS | - | Bulk import in progress |
| Pod OOMKilled | K8s | - | Memory limit too low |
| Pod Pending | K8s | - | Insufficient cluster capacity |
| CrashLoopBackOff | K8s | - | Config error |
| Node NotReady | K8s | EC2 | Disk pressure on EC2 |
| App Slow | EC2 | RDS | Missing database index |
| Cascading Failure | K8s | RDS | Blocking database query |

---

*Last Updated: 2026-01-18*
