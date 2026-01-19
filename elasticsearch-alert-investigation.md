---
name: elasticsearch-alert-investigation
description: This skill should be used when the user asks to "investigate Elasticsearch alert", "debug ES cluster", "check OpenSearch health", "analyze search performance", mentions Elasticsearch/OpenSearch issues, cluster health red/yellow, shard allocation problems, JVM memory pressure, or receives alerts about search clusters including index issues, query latency, or storage problems.
version: 2.0.0
---

# Elasticsearch/OpenSearch Alert Investigation Skill v2.0

## Overview

Comprehensive protocol for investigating Elasticsearch/OpenSearch cluster alerts with cluster health analysis, shard diagnostics, and performance troubleshooting.

## Quick Reference

| Data Source | UID/Server | Purpose |
|-------------|------------|---------|
| Prometheus | `df8o21agxtkw0d` (UMBQuerier-Luckin) | ES exporter metrics |
| CloudWatch | ES namespace | AWS OpenSearch metrics |
| DevOps DB | `aws-luckyus-devops-rw` | Service topology |

## Alert Categories

| Category | Trigger Conditions | Priority |
|----------|-------------------|----------|
| Cluster Health | Status RED or YELLOW | L0 (RED) / L1 (YELLOW) |
| CPU | > 80% sustained | L1/L2 |
| JVM Memory | Heap > 85% | L0/L1 |
| Storage | < 15% free OR < 10GB | L0/L1 |
| Shards | Unassigned shards > 0 | L1 |
| Query Latency | p99 > 500ms | L1/L2 |

## Investigation Protocol

### Phase 1: Alert Validation (Max 30s)

Extract from alert payload:
- `cluster_name`, `domain_name`
- Alert type, current value, threshold
- Node count and roles

Classify cluster tier:
- **L0**: Production search (product, order search)
- **L1**: Business analytics, logging
- **L2**: Development, testing clusters

### Phase 2: Parallel Health Collection

**CRITICAL**: Execute these Prometheus queries in parallel:

```promql
# Cluster Health (parallel)
elasticsearch_cluster_health_status{cluster="$CLUSTER"}
elasticsearch_cluster_health_number_of_nodes{cluster="$CLUSTER"}
elasticsearch_cluster_health_active_shards{cluster="$CLUSTER"}
elasticsearch_cluster_health_unassigned_shards{cluster="$CLUSTER"}
elasticsearch_cluster_health_relocating_shards{cluster="$CLUSTER"}

# Resource Usage (parallel)
elasticsearch_jvm_memory_used_bytes{cluster="$CLUSTER"} / elasticsearch_jvm_memory_max_bytes{cluster="$CLUSTER"}
elasticsearch_process_cpu_percent{cluster="$CLUSTER"}
elasticsearch_filesystem_data_available_bytes{cluster="$CLUSTER"}
elasticsearch_filesystem_data_size_bytes{cluster="$CLUSTER"}
```

### Phase 3: Cluster Status Deep Dive

**Health Status Interpretation**:
- `GREEN (1)`: All primary and replica shards assigned
- `YELLOW (2)`: All primaries assigned, some replicas unassigned
- `RED (3)`: Some primary shards unassigned

**Node Health Analysis**:
```promql
# Per-node metrics
elasticsearch_jvm_gc_collection_seconds_sum{cluster="$CLUSTER"}
elasticsearch_thread_pool_rejected_count{cluster="$CLUSTER"}
elasticsearch_breakers_tripped{cluster="$CLUSTER"}
```

### Phase 4: Shard Analysis (for cluster health issues)

**Unassigned Shard Investigation**:
```promql
elasticsearch_indices_shards_docs{cluster="$CLUSTER",type="primary"}
elasticsearch_indices_store_size_bytes{cluster="$CLUSTER"}
```

Common unassigned reasons:
- `INDEX_CREATED`: New index awaiting allocation
- `CLUSTER_RECOVERED`: Recovery in progress
- `NODE_LEFT`: Node departed cluster
- `ALLOCATION_FAILED`: Disk space or node capacity
- `REPLICA_ADDED`: Adding replicas

### Phase 5: Index Performance Analysis

**Query Performance**:
```promql
# Search latency
rate(elasticsearch_indices_search_query_time_seconds_total{cluster="$CLUSTER"}[5m]) /
rate(elasticsearch_indices_search_query_total{cluster="$CLUSTER"}[5m])

# Indexing latency
rate(elasticsearch_indices_indexing_index_time_seconds_total{cluster="$CLUSTER"}[5m]) /
rate(elasticsearch_indices_indexing_index_total{cluster="$CLUSTER"}[5m])

# Request rate
rate(elasticsearch_indices_search_query_total{cluster="$CLUSTER"}[5m])
```

**Indexing Pressure**:
```promql
elasticsearch_indices_indexing_index_current{cluster="$CLUSTER"}
elasticsearch_indices_get_current{cluster="$CLUSTER"}
elasticsearch_indices_merges_current{cluster="$CLUSTER"}
```

### Phase 6: JVM Analysis (for memory alerts)

```promql
# Heap usage per pool
elasticsearch_jvm_memory_pool_used_bytes{cluster="$CLUSTER"}
elasticsearch_jvm_memory_pool_max_bytes{cluster="$CLUSTER"}

# GC metrics
rate(elasticsearch_jvm_gc_collection_seconds_sum{cluster="$CLUSTER"}[5m])
rate(elasticsearch_jvm_gc_collection_seconds_count{cluster="$CLUSTER"}[5m])
```

GC Pressure Indicators:
- Young GC > 50ms average = investigate allocation rate
- Old GC > 1s = potential memory pressure
- GC overhead > 5% = serious issue

### Phase 7: CloudWatch Integration

Query AWS OpenSearch/Elasticsearch metrics:
- `ClusterStatus.green`, `ClusterStatus.yellow`, `ClusterStatus.red`
- `CPUUtilization`
- `JVMMemoryPressure`
- `FreeStorageSpace`
- `SearchLatency`, `IndexingLatency`
- `ThreadpoolWriteRejected`, `ThreadpoolSearchRejected`

### Phase 8: Service Dependency Analysis

```sql
-- Find services using this ES cluster
SELECT s.service_name, s.service_priority,
       sem.index_patterns, sem.query_types
FROM services s
JOIN service_es_mapping sem ON s.id = sem.service_id
WHERE sem.cluster_name = '$CLUSTER';
```

## Root Cause Patterns

| Symptom Pattern | Likely Cause | Action |
|-----------------|--------------|--------|
| RED + unassigned primaries | Node failure or disk full | Check node status, expand storage |
| YELLOW + unassigned replicas | Insufficient nodes | Add nodes or reduce replicas |
| High JVM + slow GC | Heap sizing issue | Tune heap or upgrade nodes |
| High CPU + query latency | Complex queries or no caching | Optimize queries, add caching |
| Storage < 15% | Index growth or no ILM | Implement ILM, delete old indices |
| Thread rejections | Overloaded cluster | Scale cluster or rate limit |

## Cross-Skill Integration

If investigation reveals:
- **Application search timeout** → Suggest `/investigate-apm`
- **Host resource constraints** → Suggest `/investigate-ec2`
- **Downstream DB issues** → Suggest `/investigate-rds`

## Error Handling

| Failure Mode | Fallback Action |
|--------------|----------------|
| Prometheus ES exporter down | Use CloudWatch OpenSearch metrics |
| CloudWatch unavailable | Parse ES logs from CloudWatch Logs |
| Cluster API unreachable | Check security groups and network |

## Escalation Criteria

Escalate immediately if:
- Cluster status RED for > 5 minutes
- JVM memory > 95%
- Primary shard loss with data at risk
- All master nodes unavailable

## Report Template

```markdown
# Elasticsearch Investigation Report

## Alert Summary
- **Cluster**: [Cluster/Domain Name]
- **Status**: [GREEN/YELLOW/RED]
- **Tier**: [L0/L1/L2]
- **Alert Type**: [Category]

## Cluster Health
| Metric | Current | Expected | Status |
|--------|---------|----------|--------|
| Nodes | X | Y | [OK/WARN] |
| Active Shards | X | Y | [OK/WARN] |
| Unassigned Shards | X | 0 | [OK/WARN/CRIT] |

## Resource Status
| Node | CPU | JVM Heap | Disk Free | Status |
|------|-----|----------|-----------|--------|
| [node1] | X% | Y% | Z GB | [OK/WARN] |

## Performance Metrics
| Metric | Current | Threshold | Status |
|--------|---------|-----------|--------|
| Search Latency (p99) | Xms | 500ms | [OK/WARN] |
| Indexing Rate | X/s | - | - |
| Thread Rejections | X | 0 | [OK/WARN] |

## Root Cause
[Determined cause with evidence]

## Affected Services
| Service | Priority | Index Pattern | Impact |
|---------|----------|---------------|--------|
| [name] | [L0/L1] | [pattern] | [description] |

## Recommendations
1. **Immediate**: [Action]
2. **Short-term**: [Action]
3. **Long-term**: [Action]
```
