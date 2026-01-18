# Elasticsearch/OpenSearch Alert Investigation Skill

> Version: 1.0 | Edition: AWS OpenSearch/Elasticsearch Diagnosis | Platform: AWS OpenSearch Service

## Skill Trigger

Use this skill when you receive alerts related to Elasticsearch/OpenSearch clusters, including:
- **CPU Alerts**: `AWS-ES CPU 使用率大于90%`
- **Cluster Health Alerts**: `AWS-ES 集群状态Red`, `AWS-ES 集群状态Yellow`
- **Storage Alerts**: `AWS-ES磁盘空间不足10G`
- **Memory Alerts**: `JVMMemoryPressure`, `MasterJVMMemoryPressure`
- **Index Alerts**: Index write blocked, shard allocation issues

---

## Verified Data Sources

**Prometheus Datasources**:
- `UMBQuerier-Luckin` (UID: `df8o21agxtkw0d`) - ES health checks and AWS metrics

**Available Metrics**:
```promql
# ES Health Check Metric
health_check_storage_elasticsearch

# AWS ElastiCache metrics (for context)
aws_elasticache_cpuutilization_average
```

**AWS CloudWatch**:
- OpenSearch metrics: `AWS/ES` namespace
- CloudWatch alarms for ES domains

**MySQL DevOps Database**:
- Server: `aws-luckyus-devops-rw` (service topology, alert logs)

---

## Investigation Protocol

### Phase 0: Alert Validation

**Objective**: Extract and validate alert metadata

**Actions**:
1. Parse the alert payload to extract:
   - `alertname`: The specific alert type
   - `domain` or `cluster`: ES domain name
   - `severity`: Alert level (critical, warning, info)
   - `startsAt`: Alert start timestamp

2. Identify ES domain from alert:
```
# Common patterns:
# - aws-luckyus-es-domain
# - luckyus-logs-es
# - luckyus-search-es
```

3. Determine service priority level:
```sql
-- Query service level from DevOps DB
-- Server: aws-luckyus-devops-rw
SELECT service_name, level, owner, oncall_group
FROM service_registry
WHERE service_name LIKE '%elasticsearch%' OR service_name LIKE '%search%';
```

**Priority Classification**:
| Priority | Response Time | Alert Type |
|----------|---------------|------------|
| L0 | < 15 minutes | Cluster Status Red, Write Blocked |
| L1 | < 30 minutes | Cluster Status Yellow, High CPU |
| L2 | < 2 hours | Low disk warning, High JVM memory |

---

### Phase 1: Data Availability Check

**Objective**: Verify ES metrics are accessible

**Actions**:

1. **Check Prometheus ES Health Metric**:
```promql
-- Use Grafana MCP: mcp__grafana__query_prometheus
-- Datasource UID: df8o21agxtkw0d (UMBQuerier-Luckin)
health_check_storage_elasticsearch
```

2. **Check CloudWatch metrics availability** (use mcp__cloudwatch-server__get_metric_data):
```
Namespace: AWS/ES
MetricName: ClusterStatus.green
Dimensions: DomainName=<domain_name>, ClientId=<account_id>
```

**If data unavailable**: Document gap and proceed with CloudWatch as primary source.

---

### Phase 2: Cluster Health Assessment

**Objective**: Collect comprehensive ES cluster metrics

**Execute queries using mcp__cloudwatch-server__get_metric_data**:

#### 2a. Cluster Status Metrics
```
Namespace: AWS/ES
Metrics:
  - ClusterStatus.green (value=1 means healthy)
  - ClusterStatus.yellow (value=1 means degraded)
  - ClusterStatus.red (value=1 means critical)
Dimensions: DomainName=<domain_name>
Period: 60 seconds
```

**Status Interpretation**:
| Status | Meaning | Action Required |
|--------|---------|-----------------|
| Green | All primary and replica shards allocated | Normal operation |
| Yellow | All primary shards allocated, some replicas missing | Monitor, plan maintenance |
| Red | Some primary shards not allocated | **IMMEDIATE ACTION REQUIRED** |

#### 2b. CPU Metrics
```
Metrics:
  - CPUUtilization
  - MasterCPUUtilization
Thresholds:
  - Warning: > 70%
  - Critical: > 90%
```

#### 2c. Memory Metrics
```
Metrics:
  - JVMMemoryPressure (percentage)
  - MasterJVMMemoryPressure
  - FreeStorageSpace (bytes)
Thresholds:
  - JVM Memory Warning: > 80%
  - JVM Memory Critical: > 95%
  - Storage Warning: < 20GB
  - Storage Critical: < 10GB
```

#### 2d. Index and Shard Metrics
```
Metrics:
  - Shards.active
  - Shards.unassigned
  - Shards.delayedUnassigned
  - Shards.activePrimary
  - Shards.initializing
  - Shards.relocating
```

#### 2e. Request Metrics
```
Metrics:
  - SearchRate
  - IndexingRate
  - SearchLatency
  - IndexingLatency
  - 2xx (successful requests)
  - 4xx (client errors)
  - 5xx (server errors)
```

#### 2f. Node Metrics
```
Metrics:
  - Nodes (total node count)
  - MasterReachableFromNode
  - ThreadpoolWriteQueue
  - ThreadpoolSearchQueue
  - ThreadpoolBulkRejected
  - ThreadpoolSearchRejected
```

---

### Phase 3: Root Cause Analysis by Alert Type

**Objective**: Targeted investigation based on alert category

#### 3a. Cluster Status Red Investigation

**Check for unassigned shards**:
```
Metrics to check:
- Shards.unassigned > 0
- Shards.activePrimary < expected
```

**Common Causes**:
1. **Node failure**: Check `Nodes` count vs expected
2. **Disk space exhaustion**: Check `FreeStorageSpace`
3. **JVM memory pressure**: Check `JVMMemoryPressure`
4. **Network issues**: Check `MasterReachableFromNode`

**Recovery Actions**:
| Cause | Immediate Action |
|-------|------------------|
| Node failure | Scale up cluster, wait for recovery |
| Disk full | Delete old indices, increase storage |
| JVM pressure | Reduce indexing rate, increase heap |

#### 3b. Cluster Status Yellow Investigation

**Check for missing replicas**:
```
Metrics to check:
- Shards.unassigned > 0 (but Shards.activePrimary = expected)
- Node count might be < replica count + 1
```

**Common Causes**:
1. **Insufficient nodes for replicas**
2. **Node temporarily unavailable**
3. **Shard allocation disabled**
4. **Index settings mismatch**

#### 3c. High CPU Investigation

**Check workload metrics**:
```
Metrics to check:
- SearchRate (too many searches?)
- IndexingRate (too much indexing?)
- ThreadpoolSearchQueue (queries backing up?)
- ThreadpoolWriteQueue (writes backing up?)
```

**Common Causes**:
1. **Heavy search workload**: Optimize queries, add nodes
2. **Heavy indexing**: Reduce bulk size, add nodes
3. **Complex aggregations**: Review query patterns
4. **Garbage collection**: Check JVMMemoryPressure

#### 3d. Disk Space Investigation

**Check storage metrics**:
```
Metrics to check:
- FreeStorageSpace
- ClusterUsedSpace
- DeletedDocuments (tombstones consuming space)
```

**Common Causes**:
1. **Index growth**: Set up ILM (Index Lifecycle Management)
2. **No retention policy**: Delete old indices
3. **Too many replicas**: Reduce replica count
4. **Deleted docs not purged**: Force merge

---

### Phase 4: Service Dependency Analysis

**Objective**: Identify applications using this ES cluster

**Actions**:

1. **Query service-to-ES mapping**:
```sql
-- Server: aws-luckyus-devops-rw
SELECT service_name, es_domain, index_pattern, purpose
FROM service_es_mapping
WHERE es_domain LIKE '%<domain_name>%';
```

2. **Common ES Use Cases**:

| Use Case | Typical Domain | Services |
|----------|----------------|----------|
| Application Logs | luckyus-logs-es | All services (ELK stack) |
| Product Search | luckyus-search-es | isales-commodity, shop |
| Analytics | luckyus-analytics-es | bigdata, reporting |
| APM Data | luckyus-apm-es | skywalking, iZeus |

3. **Check dependent service health**:
```promql
-- Datasource UID: df8o21agxtkw0d
up{job=~".*<service_name>.*"}
```

---

### Phase 5: Alert History Analysis

**Objective**: Analyze historical alert patterns

**Actions**:

1. **Query alert history from DevOps DB**:
```sql
-- Server: aws-luckyus-devops-rw
SELECT alert_name, alert_status, instance, severity, create_time
FROM t_umb_alert_log
WHERE alert_name LIKE '%ES%' OR alert_name LIKE '%Elasticsearch%'
ORDER BY create_time DESC
LIMIT 50;
```

2. **Identify recurring patterns**:
```sql
SELECT DATE(create_time) as alert_date, alert_name, COUNT(*) as count
FROM t_umb_alert_log
WHERE alert_name LIKE '%ES%'
  AND create_time > DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY DATE(create_time), alert_name
ORDER BY alert_date DESC, count DESC;
```

3. **Check for correlated alerts**:
```sql
SELECT alert_name, COUNT(*) as count
FROM t_umb_alert_log
WHERE create_time BETWEEN '<alert_start - 30min>' AND '<alert_start + 30min>'
GROUP BY alert_name
ORDER BY count DESC;
```

---

### Phase 6: Output Report Generation

**Objective**: Generate structured investigation report

**Report Template**:

```markdown
## Elasticsearch/OpenSearch Investigation Report

### Alert Summary
| Field | Value |
|-------|-------|
| Alert Name | [alertname] |
| ES Domain | [domain_name] |
| Severity | [L0/L1/L2] |
| Start Time | [timestamp] |
| Duration | [duration] |

### Service Context
- **Service Level**: [L0/L1/L2]
- **Dependent Services**: [service_list]
- **Service Owner**: [owner]
- **On-Call Group**: [oncall_group]

### Investigation Findings

#### Phase 1: Data Availability
- CloudWatch Metrics: [Available/Unavailable]
- Prometheus Health Check: [Available/Unavailable]

#### Phase 2: Cluster Health
| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| Cluster Status | [Green/Yellow/Red] | Green | [OK/WARNING/CRITICAL] |
| CPU Utilization | [X%] | 90% | [OK/WARNING/CRITICAL] |
| JVM Memory Pressure | [X%] | 95% | [OK/WARNING/CRITICAL] |
| Free Storage | [X GB] | 10GB | [OK/WARNING/CRITICAL] |
| Active Shards | [X] | [expected] | [OK/WARNING/CRITICAL] |
| Unassigned Shards | [X] | 0 | [OK/WARNING/CRITICAL] |
| Node Count | [X] | [expected] | [OK/WARNING/CRITICAL] |

#### Phase 3: Root Cause Analysis
**Alert Type**: [Cluster Red/Yellow/High CPU/Disk Full]

**Findings**:
- [Detailed findings based on alert type]

**Immediate Cause**: [Identified cause]

#### Phase 4: Dependent Services
| Service | Status | Impact |
|---------|--------|--------|
| [service1] | [Running/Degraded] | [Description] |
| [service2] | [Running/Degraded] | [Description] |

#### Phase 5: Alert History
- **Recent Alert Count**: [X in last 24h]
- **Pattern Detected**: [Yes/No - describe pattern]
- **Correlated Alerts**: [list]

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
<summary>CloudWatch Metrics</summary>

[Include CloudWatch metric data]

</details>

<details>
<summary>Alert History</summary>

[Include alert history from DevOps DB]

</details>

---
*Investigation completed at [timestamp]*
*Skill Version: Elasticsearch Alert Investigation SOP v1.0*
```

---

## MCP Tools Reference

| Tool | Server | Purpose |
|------|--------|---------|
| `mcp__grafana__query_prometheus` | grafana | ES health check metrics |
| `mcp__mcp-db-gateway__mysql_query` | mcp-db-gateway | Service topology, alert logs |
| `mcp__cloudwatch-server__get_metric_data` | cloudwatch-server | AWS OpenSearch metrics |
| `mcp__cloudwatch-server__get_active_alarms` | cloudwatch-server | Related alarms |

## Key Datasource UIDs

| Datasource | UID | Use |
|------------|-----|-----|
| UMBQuerier-Luckin | `df8o21agxtkw0d` | ES health checks, AWS metrics |

## DevOps Database

| Server | Purpose |
|--------|---------|
| `aws-luckyus-devops-rw` | Service registry, alert logs, ES mappings |

---

## Common ES Alert Scenarios

### Scenario 1: Cluster Status Red
**Symptoms**: ClusterStatus.red = 1, unassigned primary shards
**Investigation Focus**:
- Check node count (node failure?)
- Check disk space (disk full causing write block?)
- Check JVM memory (GC pressure?)
- Review recent changes (index settings?)

**Emergency Actions**:
1. Scale up cluster nodes if node failure
2. Delete old indices if disk full
3. Increase JVM heap if memory pressure
4. Disable new indexing temporarily if needed

### Scenario 2: Cluster Status Yellow
**Symptoms**: ClusterStatus.yellow = 1, unassigned replica shards
**Investigation Focus**:
- Check if node count < replica factor + 1
- Check for node maintenance window
- Review shard allocation settings

### Scenario 3: High CPU (> 90%)
**Symptoms**: CPUUtilization > 90% sustained
**Investigation Focus**:
- Check search rate and latency
- Check indexing rate
- Look for complex aggregations
- Review query patterns

### Scenario 4: Disk Space Critical (< 10GB)
**Symptoms**: FreeStorageSpace < 10GB
**Investigation Focus**:
- Identify largest indices
- Check index lifecycle policy
- Look for deleted but not purged documents
- Review replica settings

---

## Example Usage

When investigating an ES alert, invoke this skill with the alert payload:

```
/investigate-elasticsearch {
  "alertname": "AWS-ES 集群状态Red",
  "domain": "luckyus-logs-es",
  "severity": "critical"
}
```

The skill will execute all 6 phases automatically and generate a comprehensive report.
