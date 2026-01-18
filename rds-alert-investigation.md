# RDS Alert Investigation Skill

> Version: 2.0 | Edition: RDS Database Diagnosis | Platform: AWS RDS (MySQL/PostgreSQL)

## Skill Trigger

Use this skill when you receive alerts related to RDS databases, including:
- **Connection Alerts**: `DatabaseConnections`, connection pool exhaustion, max connections
- **Slow Query Alerts**: `SlowQueries`, query performance degradation, long-running queries
- **Storage Alerts**: `FreeStorageSpace`, `FreeableMemory`, storage capacity warnings
- **Replication Alerts**: `ReplicaLag`, replication failures, sync issues
- **CPU Alerts**: `CPUUtilization`, high CPU from expensive queries

---

## Verified Data Sources

**MySQL Servers** (61 available via mcp-db-gateway):
- Core: `aws-luckyus-salesorder-rw`, `aws-luckyus-payment-rw`, `aws-luckyus-user-rw`
- DevOps: `aws-luckyus-devops-rw` (service registry and topology)
- Inventory: `aws-luckyus-inventory-rw`, `aws-luckyus-itemscm-rw`
- Analytics: `aws-luckyus-datacenter-rw`, `aws-luckyus-analysis-rw`

**PostgreSQL Servers** (3 available):
- `aws-luckyus-dify-rw`, `aws-luckyus-difynew-rw`, `aws-luckyus-pgilkmap-rw`

**CloudWatch Slow Query Logs** (64 RDS instances):
- Pattern: `/aws/rds/instance/<instance-name>/slowquery`
- Key instances: `aws-luckyus-salesorder`, `aws-luckyus-order`, `aws-luckyus-payment`

**Redis Clusters** (74 available):
- For cache dependency checks

---

## Investigation Protocol

### Phase 0: Alert Validation

**Objective**: Parse alert and identify RDS instance details

**Actions**:
1. Parse the alert payload to extract:
   - `alertname`: The specific alert type
   - `db_instance`: RDS instance identifier
   - `db_engine`: MySQL or PostgreSQL
   - `severity`: Alert level
   - `service_name`: Associated application service
   - `startsAt`: Alert start timestamp

2. Identify the correct database server from MCP configuration:
```
-- Use mcp__mcp-db-gateway__list_servers to find the server
-- MySQL servers: 61 available
-- PostgreSQL servers: 3 available (aws-luckyus-dify-rw, aws-luckyus-difynew-rw, aws-luckyus-pgilkmap-rw)
```

3. Determine service priority by querying DevOps database:
```sql
-- Server: aws-luckyus-devops-rw
SELECT db_instance, service_name, level, owner, oncall_group
FROM rds_service_mapping
WHERE db_instance = '<rds_instance>';
```

**Priority Classification**:
| Priority | Response Time | Examples |
|----------|---------------|----------|
| L0 | < 15 minutes | Core transaction DBs, Order DB, Payment DB |
| L1 | < 30 minutes | Reporting DBs, Inventory DB |
| L2 | < 2 hours | Analytics DBs, Dev/Test DBs |

---

### Phase 1: Data Availability Check

**Objective**: Verify database connectivity and monitoring status

**Actions**:

1. **Test Database Connectivity**:
```sql
-- Use mcp__mcp-db-gateway__mysql_query or mcp__mcp-db-gateway__postgres_query
-- Server: <identified_rds_server>
SELECT 1 AS connectivity_test;
```

2. **Check Database Version and Status**:

For MySQL:
```sql
SHOW VARIABLES LIKE 'version%';
SHOW STATUS LIKE 'Uptime';
SELECT @@hostname, @@read_only;
```

For PostgreSQL:
```sql
SELECT version();
SELECT pg_postmaster_start_time();
SELECT pg_is_in_recovery();
```

3. **Verify Prometheus Exporter** (if available):
```promql
-- Use mcp__grafana__query_prometheus
-- Datasource UID: df8o21agxtkw0d
mysql_up{instance=~"<rds_endpoint>.*"}
-- or for PostgreSQL
pg_up{instance=~"<rds_endpoint>.*"}
```

---

### Phase 2: Database Health Analysis

**Objective**: Comprehensive database health investigation based on alert type

#### Phase 2a: Connection Analysis

**For Connection-related alerts**:

MySQL:
```sql
-- Current connections
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
SHOW VARIABLES LIKE 'max_connections';

-- Connection breakdown by user/host
SELECT user, host, COUNT(*) as connections
FROM information_schema.processlist
GROUP BY user, host
ORDER BY connections DESC;

-- Active vs idle connections
SELECT
    SUM(CASE WHEN command = 'Sleep' THEN 1 ELSE 0 END) as idle,
    SUM(CASE WHEN command != 'Sleep' THEN 1 ELSE 0 END) as active
FROM information_schema.processlist;

-- Connections by state
SELECT state, COUNT(*) as count
FROM information_schema.processlist
WHERE command != 'Sleep'
GROUP BY state
ORDER BY count DESC;
```

PostgreSQL:
```sql
-- Current connections
SELECT count(*) as total_connections FROM pg_stat_activity;

-- Max connections
SHOW max_connections;

-- Connections by state
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;

-- Connections by application
SELECT application_name, count(*)
FROM pg_stat_activity
GROUP BY application_name
ORDER BY count DESC;

-- Waiting connections
SELECT count(*) as waiting FROM pg_stat_activity WHERE wait_event IS NOT NULL;
```

#### Phase 2b: Slow Query Investigation

**For Slow Query alerts**:

MySQL:
```sql
-- Check slow query log status
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';

-- Current running queries > 5 seconds
SELECT id, user, host, db, command, time, state,
       LEFT(info, 200) as query
FROM information_schema.processlist
WHERE command != 'Sleep' AND time > 5
ORDER BY time DESC;

-- Lock contention
SELECT * FROM information_schema.innodb_lock_waits;

-- Table lock waits
SHOW STATUS LIKE 'Table_locks_waited';
SHOW STATUS LIKE 'Table_locks_immediate';
```

Also query CloudWatch slow query logs:
```
-- Use mcp__cloudwatch-server__execute_log_insights_query
-- Log group: /aws/rds/instance/<rds_instance>/slowquery
fields @timestamp, @message
| parse @message "# Query_time: * Lock_time: * Rows_sent: * Rows_examined: *" as query_time, lock_time, rows_sent, rows_examined
| filter query_time > 1
| sort query_time desc
| limit 20
```

PostgreSQL:
```sql
-- Long running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 seconds'
AND state != 'idle'
ORDER BY duration DESC;

-- Lock contention
SELECT blocked_locks.pid AS blocked_pid,
       blocked_activity.usename AS blocked_user,
       blocking_locks.pid AS blocking_pid,
       blocking_activity.usename AS blocking_user,
       blocked_activity.query AS blocked_statement,
       blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

#### Phase 2c: Storage Analysis

**For Storage alerts**:

MySQL:
```sql
-- Database sizes
SELECT table_schema as db_name,
       ROUND(SUM(data_length + index_length) / 1024 / 1024 / 1024, 2) as size_gb
FROM information_schema.tables
GROUP BY table_schema
ORDER BY size_gb DESC;

-- Largest tables
SELECT table_schema, table_name,
       ROUND((data_length + index_length) / 1024 / 1024 / 1024, 2) as size_gb,
       table_rows
FROM information_schema.tables
ORDER BY (data_length + index_length) DESC
LIMIT 20;

-- InnoDB buffer pool usage
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- Binary log space
SHOW BINARY LOGS;
```

PostgreSQL:
```sql
-- Database sizes
SELECT pg_database.datname as db_name,
       pg_size_pretty(pg_database_size(pg_database.datname)) as size
FROM pg_database
ORDER BY pg_database_size(pg_database.datname) DESC;

-- Largest tables
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) as total_size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;

-- Bloat estimation
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) as total_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 10;
```

#### Phase 2d: Replication Status

**For Replication alerts**:

MySQL (for replicas):
```sql
-- Replica status
SHOW REPLICA STATUS\G

-- Key metrics to check:
-- Seconds_Behind_Master (Replica_Lag)
-- Slave_IO_Running
-- Slave_SQL_Running
-- Last_Error
```

PostgreSQL (for replicas):
```sql
-- Check if this is a replica
SELECT pg_is_in_recovery();

-- Replication lag (on primary)
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       (extract(epoch from now()) - extract(epoch from reply_time))::int as lag_seconds
FROM pg_stat_replication;

-- WAL receiver status (on replica)
SELECT * FROM pg_stat_wal_receiver;
```

#### Phase 2e: CPU Analysis

**For CPU alerts**:

MySQL:
```sql
-- Top CPU consumers (queries)
SELECT id, user, host, db, command, time, state,
       LEFT(info, 200) as query
FROM information_schema.processlist
WHERE command NOT IN ('Sleep', 'Binlog Dump')
ORDER BY time DESC
LIMIT 20;

-- Status counters
SHOW GLOBAL STATUS LIKE 'Questions';
SHOW GLOBAL STATUS LIKE 'Queries';
SHOW GLOBAL STATUS LIKE 'Com_select';
SHOW GLOBAL STATUS LIKE 'Com_insert';
SHOW GLOBAL STATUS LIKE 'Com_update';
SHOW GLOBAL STATUS LIKE 'Com_delete';
```

---

### Phase 3: Related Cache Analysis

**Objective**: Check Redis cache health for applications using this database

**Actions**:

1. **Identify related Redis clusters** from DevOps DB:
```sql
-- Server: aws-luckyus-devops-rw
SELECT redis_cluster, cache_purpose
FROM service_cache_mapping
WHERE db_instance = '<rds_instance>';
```

2. **Check Redis health**:
```
-- Use mcp__mcp-db-gateway__redis_command
INFO memory
INFO clients
INFO stats
INFO replication
```

3. **Query Redis metrics**:
```promql
-- Datasource UID: ff6p0gjt24phce (prometheus_redis)
redis_connected_clients{instance=~"<redis_cluster>.*"}
redis_memory_used_bytes{instance=~"<redis_cluster>.*"}
rate(redis_commands_processed_total{instance=~"<redis_cluster>.*"}[5m])
redis_keyspace_hits_total{instance=~"<redis_cluster>.*"}
redis_keyspace_misses_total{instance=~"<redis_cluster>.*"}
```

4. **Check cache hit rate**:
```promql
rate(redis_keyspace_hits_total{instance=~"<redis_cluster>.*"}[5m]) /
(rate(redis_keyspace_hits_total{instance=~"<redis_cluster>.*"}[5m]) + rate(redis_keyspace_misses_total{instance=~"<redis_cluster>.*"}[5m]))
```

---

### Phase 4: Related Service Analysis

**Objective**: Check EC2 instances and services connecting to this database

**Actions**:

1. **Identify dependent services**:
```sql
-- Server: aws-luckyus-devops-rw
SELECT service_name, instance_group, connection_pool_size
FROM service_db_mapping
WHERE db_instance = '<rds_instance>';
```

2. **Check application server health**:
```promql
-- Datasource UID: df8o21agxtkw0d
up{job=~"<service_name>.*"}

-- Memory usage of dependent services
100 - ((node_memory_MemAvailable_bytes{job=~"<service_name>.*"} / node_memory_MemTotal_bytes{job=~"<service_name>.*"}) * 100)

-- Connection pool metrics (if available)
hikaricp_connections_active{application=~"<service_name>.*"}
hikaricp_connections_pending{application=~"<service_name>.*"}
```

---

### Phase 5: CloudWatch Integration

**Objective**: Gather AWS RDS-specific CloudWatch metrics

**Actions**:

1. **Get RDS CloudWatch Metrics** (use mcp__cloudwatch-server__get_metric_data):
```
Namespace: AWS/RDS
Dimensions: DBInstanceIdentifier=<rds_instance>
Metrics to collect:
- CPUUtilization
- DatabaseConnections
- FreeableMemory
- FreeStorageSpace
- ReadIOPS
- WriteIOPS
- ReadLatency
- WriteLatency
- ReplicaLag (if replica)
- NetworkReceiveThroughput
- NetworkTransmitThroughput
```

2. **Query RDS Logs** from CloudWatch:
```
-- Log groups available:
-- /aws/rds/instance/<instance>/slowquery
-- /aws/rds/instance/<instance>/error
-- /aws/rds/instance/<instance>/general

-- Use mcp__cloudwatch-server__execute_log_insights_query
fields @timestamp, @message
| filter @message like /error|Error|ERROR|warning|Warning/
| sort @timestamp desc
| limit 50
```

3. **Check Performance Insights** (if enabled):
```
-- Use mcp__cloudwatch-server__get_metric_data
Namespace: AWS/RDS
Metrics: DBLoad, DBLoadCPU, DBLoadNonCPU
```

---

### Phase 6: Alert Correlation

**Objective**: Find related alerts from the same time window

**Actions**:

1. **Query recent alerts** from alert log:
```sql
-- Server: aws-luckyus-devops-rw
SELECT alert_name, instance, severity, starts_at, ends_at, labels
FROM alert_log
WHERE starts_at >= NOW() - INTERVAL 2 HOUR
  AND (
    labels LIKE '%<rds_instance>%'
    OR labels LIKE '%<service_name>%'
  )
ORDER BY starts_at DESC;
```

2. **Check for cascade patterns**:
- Did application alerts precede database alerts?
- Are there connection exhaustion patterns?
- Are multiple services affected?

---

### Phase 7: Output Report Generation

**Objective**: Generate comprehensive database investigation report

**Report Template**:

```markdown
## RDS Investigation Report

### Alert Summary
| Field | Value |
|-------|-------|
| Alert Name | [alertname] |
| RDS Instance | [db_instance] |
| DB Engine | [MySQL/PostgreSQL] |
| Engine Version | [version] |
| Service | [service_name] |
| Severity | [L0/L1/L2] |
| Start Time | [timestamp] |
| Duration | [duration] |

### Service Context
- **Service Level**: [L0/L1/L2]
- **Dependent Services**: [list]
- **Service Owner**: [owner]
- **On-Call Group**: [oncall_group]

### Database Health Status

#### Connection Analysis
| Metric | Current | Max | Status |
|--------|---------|-----|--------|
| Total Connections | [X] | [max] | [OK/WARNING/CRITICAL] |
| Active Queries | [X] | - | [OK/WARNING] |
| Idle Connections | [X] | - | [OK/WARNING] |
| Waiting Connections | [X] | - | [OK/CRITICAL] |

**Top Connection Sources**:
| User | Host | Connections |
|------|------|-------------|
| [user1] | [host1] | [count] |
| [user2] | [host2] | [count] |

#### Query Performance
| Metric | Value | Status |
|--------|-------|--------|
| Long Running Queries | [count] | [OK/WARNING/CRITICAL] |
| Lock Waits | [count] | [OK/CRITICAL] |
| Slow Query Rate | [X/min] | [OK/WARNING] |

**Problematic Queries** (if any):
```sql
[Query 1 - execution time, rows examined]
[Query 2 - execution time, rows examined]
```

#### Storage Status
| Metric | Current | Total | Status |
|--------|---------|-------|--------|
| Free Storage | [X GB] | [Y GB] | [OK/WARNING/CRITICAL] |
| Freeable Memory | [X GB] | [Y GB] | [OK/WARNING/CRITICAL] |

**Largest Tables**:
| Database | Table | Size |
|----------|-------|------|
| [db1] | [table1] | [X GB] |
| [db2] | [table2] | [X GB] |

#### Replication Status (if applicable)
| Metric | Value | Status |
|--------|-------|--------|
| Replica Lag | [X seconds] | [OK/WARNING/CRITICAL] |
| IO Thread | [Running/Stopped] | [OK/CRITICAL] |
| SQL Thread | [Running/Stopped] | [OK/CRITICAL] |
| Last Error | [error or None] | - |

#### CPU Analysis
| Metric | Value | Status |
|--------|-------|--------|
| CPU Utilization | [X%] | [OK/WARNING/CRITICAL] |
| Query Rate | [X/sec] | [Normal/High] |

### Dependencies Status

#### Cache (Redis)
| Cluster | Memory | Connections | Hit Rate | Status |
|---------|--------|-------------|----------|--------|
| [cluster] | [X%] | [Y] | [Z%] | [Healthy/Issues] |

#### Application Servers
| Service | Instances | Health | DB Connections |
|---------|-----------|--------|----------------|
| [service1] | [X] | [Healthy/Unhealthy] | [Y] |

### CloudWatch Correlation
| Metric | 1h Avg | Current | Trend |
|--------|--------|---------|-------|
| CPUUtilization | [X%] | [Y%] | [Stable/Increasing/Decreasing] |
| DatabaseConnections | [X] | [Y] | [Stable/Increasing/Decreasing] |
| FreeableMemory | [X GB] | [Y GB] | [Stable/Decreasing] |
| ReadLatency | [X ms] | [Y ms] | [Stable/Increasing] |
| WriteLatency | [X ms] | [Y ms] | [Stable/Increasing] |

### Related Alerts
| Alert | Instance | Time | Status |
|-------|----------|------|--------|
| [alert1] | [instance] | [time] | [Firing/Resolved] |

### Root Cause Analysis
**Primary Cause**: [Identified root cause]
**Contributing Factors**: [Additional factors]
**Impact Assessment**: [Scope and severity of impact]

### Recommendations
| Priority | Action | Owner | Timeline |
|----------|--------|-------|----------|
| P0 | [Immediate action] | [team] | Immediate |
| P1 | [Short-term fix] | [team] | 24 hours |
| P2 | [Long-term solution] | [team] | 1 week |

### Evidence

<details>
<summary>Database Query Results</summary>

[Include actual query outputs]

</details>

<details>
<summary>CloudWatch Metrics</summary>

[Include CloudWatch data]

</details>

<details>
<summary>Slow Query Log Analysis</summary>

[Include slow query analysis]

</details>

---
*Investigation completed at [timestamp]*
*Skill Version: RDS Alert Investigation SOP v1.0*
```

---

## MCP Tools Reference

| Tool | Server | Purpose |
|------|--------|---------|
| `mcp__mcp-db-gateway__mysql_query` | mcp-db-gateway | MySQL database queries (60 servers) |
| `mcp__mcp-db-gateway__postgres_query` | mcp-db-gateway | PostgreSQL queries (3 servers) |
| `mcp__mcp-db-gateway__redis_command` | mcp-db-gateway | Related cache analysis |
| `mcp__mcp-db-gateway__list_servers` | mcp-db-gateway | List available DB servers |
| `mcp__grafana__query_prometheus` | grafana | Database exporter metrics |
| `mcp__cloudwatch-server__get_metric_data` | cloudwatch-server | RDS CloudWatch metrics |
| `mcp__cloudwatch-server__execute_log_insights_query` | cloudwatch-server | RDS log analysis |
| `mcp__cloudwatch-server__describe_log_groups` | cloudwatch-server | Find RDS log groups |

## Key Database Servers

| Server | Type | Purpose |
|--------|------|---------|
| `aws-luckyus-devops-rw` | MySQL | DevOps database (service registry, alert logs) |
| `aws-luckyus-dify-rw` | PostgreSQL | Dify application DB |
| `aws-luckyus-difynew-rw` | PostgreSQL | Dify new application DB |
| `aws-luckyus-pgilkmap-rw` | PostgreSQL | Map service DB |

## CloudWatch Log Groups for RDS

Available slow query log groups:
- `/aws/rds/instance/aws-luckyus-*/slowquery` (40+ databases)

---

## Example Usage

When investigating an RDS alert, invoke this skill with the alert payload:

```
/investigate-rds {
  "alertname": "DatabaseConnections",
  "db_instance": "aws-luckyus-order-rw",
  "db_engine": "mysql",
  "severity": "critical",
  "service_name": "order-service"
}
```

The skill will execute all 8 phases automatically and generate a comprehensive database investigation report.
