---
name: rds-alert-investigation
description: This skill should be used when the user asks to "investigate RDS alert", "debug database issues", "check RDS performance", "analyze MySQL/PostgreSQL problems", mentions database connection issues, slow queries, replication lag, or receives alerts about AWS RDS instances including connection exhaustion, query performance, storage, or CPU spikes.
version: 3.0.0
---

# RDS Alert Investigation Skill v3.0

## Overview

Comprehensive protocol for investigating AWS RDS database alerts with parallel diagnostics, query analysis, and dependency correlation.

## Quick Reference

| Data Source | Server/UID | Purpose |
|-------------|------------|---------|
| MySQL Servers | 61 available | Direct query access |
| PostgreSQL | 3 servers | Direct query access |
| Prometheus | `df8o21agxtkw0d` | Database metrics |
| CloudWatch | RDS namespace | AWS-native metrics |
| DevOps DB | `aws-luckyus-devops-rw` | Service topology |

## Alert Categories

| Category | Trigger Conditions | Priority |
|----------|-------------------|----------|
| Connections | > 80% max OR connection errors | L0/L1 |
| Query Performance | Slow queries > 10s OR deadlocks | L1/L2 |
| Replication | Lag > 30s OR slave stopped | L0 |
| Storage | < 10% free OR growth > 5GB/day | L1 |
| CPU | > 80% sustained | L1/L2 |

## Investigation Protocol

### Phase 1: Alert Validation (Max 30s)

Extract from alert payload:
- `db_instance_id`, `endpoint`, `engine` (MySQL/PostgreSQL)
- Alert type, current value, threshold
- Affected database/schema if applicable

Classify database tier:
- **Core**: order_db, payment_db, user_db (L0)
- **Business**: inventory_db, promotion_db (L1)
- **Supporting**: analytics_db, log_db (L2)

### Phase 2: Parallel Data Collection

**CRITICAL**: Execute these in parallel:

**MySQL Connection Analysis**:
```sql
-- Active connections by user/host
SELECT user, host, COUNT(*) as connections,
       SUM(IF(command='Sleep', 1, 0)) as idle
FROM information_schema.processlist
GROUP BY user, host ORDER BY connections DESC LIMIT 20;

-- Connection state breakdown
SELECT command, state, COUNT(*) as count
FROM information_schema.processlist
GROUP BY command, state ORDER BY count DESC;
```

**PostgreSQL Connection Analysis**:
```sql
SELECT usename, client_addr, state, COUNT(*) as connections
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY usename, client_addr, state ORDER BY connections DESC;
```

### Phase 3: Query Performance Analysis

**MySQL Long Queries**:
```sql
SELECT id, user, host, db, time, state,
       LEFT(info, 200) as query_preview
FROM information_schema.processlist
WHERE command != 'Sleep' AND time > 5
ORDER BY time DESC LIMIT 10;
```

**MySQL Lock Detection**:
```sql
SELECT r.trx_id waiting_trx, r.trx_mysql_thread_id waiting_thread,
       b.trx_id blocking_trx, b.trx_mysql_thread_id blocking_thread
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;
```

**PostgreSQL Long Queries**:
```sql
SELECT pid, usename, query_start, state,
       LEFT(query, 200) as query_preview,
       EXTRACT(EPOCH FROM now() - query_start) as duration_sec
FROM pg_stat_activity
WHERE state = 'active' AND query_start < now() - interval '5 seconds'
ORDER BY query_start LIMIT 10;
```

### Phase 4: Storage Analysis

**MySQL Table Sizes**:
```sql
SELECT table_schema, table_name,
       ROUND(data_length/1024/1024, 2) as data_mb,
       ROUND(index_length/1024/1024, 2) as index_mb,
       table_rows
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql','information_schema','performance_schema')
ORDER BY data_length + index_length DESC LIMIT 20;
```

**PostgreSQL Table Bloat**:
```sql
SELECT schemaname, relname,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) as total_size,
       n_dead_tup, n_live_tup,
       ROUND(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) as dead_ratio
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 20;
```

### Phase 5: Replication Health

**MySQL Replication**:
```sql
SHOW SLAVE STATUS\G
-- Check: Seconds_Behind_Master, Slave_IO_Running, Slave_SQL_Running
```

**PostgreSQL Replication**:
```sql
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn,
       pg_wal_lsn_diff(sent_lsn, flush_lsn) as lag_bytes
FROM pg_stat_replication;
```

### Phase 6: Dependency Impact

```sql
-- Find services depending on this database
SELECT s.service_name, s.service_priority, s.instance_count
FROM services s
JOIN service_db_mapping sdm ON s.id = sdm.service_id
WHERE sdm.db_host = '$DB_ENDPOINT';
```

Check dependent service health via Prometheus:
```promql
http_requests_total{service=~"$DEPENDENT_SERVICES"}
rate(http_request_duration_seconds_sum{service=~"$DEPENDENT_SERVICES"}[5m])
```

### Phase 7: CloudWatch Correlation

Query AWS RDS metrics:
- `DatabaseConnections`
- `CPUUtilization`
- `ReadIOPS`, `WriteIOPS`
- `FreeStorageSpace`
- `ReplicaLag`

Check for RDS events in the past 24h.

## Cross-Skill Integration

If investigation reveals:
- **Application connection pool issues** → Suggest `/investigate-apm`
- **Cache miss causing DB load** → Suggest `/investigate-redis`
- **Host resource issues** → Suggest `/investigate-ec2`

## Root Cause Patterns

| Symptom Pattern | Likely Cause | Action |
|-----------------|--------------|--------|
| Connections spike + slow queries | Missing indexes | EXPLAIN ANALYZE top queries |
| Replication lag + high CPU | Write-heavy workload | Consider read replicas |
| Storage growth + large tables | Audit logs or temp data | Archive or partition |
| Lock waits + long transactions | Transaction not committed | Find and kill blocking query |

## Error Handling

| Failure Mode | Fallback Action |
|--------------|----------------|
| Direct DB access denied | Use CloudWatch metrics only |
| Read replica unavailable | Query primary (with caution) |
| Prometheus gaps | Cross-reference with CloudWatch |

## Escalation Criteria

Escalate immediately if:
- Primary database unreachable
- Replication completely stopped
- > 90% connections exhausted
- Data corruption indicators

## Report Template

```markdown
# RDS Investigation Report

## Alert Summary
- **Instance**: [DB Instance ID]
- **Engine**: [MySQL/PostgreSQL] [Version]
- **Tier**: [Core/Business/Supporting]
- **Alert Type**: [Category]

## Connection Status
| Metric | Current | Max | Utilization |
|--------|---------|-----|-------------|
| Connections | X | Y | Z% |
| Active Queries | N | - | - |

## Top Resource Consumers
1. [User/Query consuming most resources]
2. [Second highest]

## Root Cause
[Determined cause with query evidence]

## Recommendations
1. **Immediate**: [Kill query X / Increase connections]
2. **Short-term**: [Add index / Optimize query]
3. **Long-term**: [Sharding / Read replica]

## Affected Services
- [Service 1]: [Impact level]
- [Service 2]: [Impact level]
```
