# MCP Data Sources Inventory

> Generated: 2026-01-18 | Status: All systems operational

## Executive Summary

| Data Source Type | Count | Status |
|------------------|-------|--------|
| **MySQL Servers** | 61 | ✅ Connected |
| **Redis Clusters** | 74 | ✅ Connected |
| **PostgreSQL Servers** | 3 | ✅ Connected |
| **Prometheus Datasources** | 3 | ✅ Connected |
| **CloudWatch Log Groups** | 64+ | ✅ Connected |
| **EKS Clusters** | 1 | ✅ Connected |
| **Elasticsearch** | 1 | ✅ Connected |

**Total Endpoints: 200+**

---

## 1. MySQL Servers (61 Total)

### MCP Server: `mcp-db-gateway`

| Server Name | Status | Primary Use |
|-------------|--------|-------------|
| `aws-luckyus-devops-rw` | ✅ Connected | DevOps service registry, alert logs |
| `aws-luckyus-salesorder-rw` | ✅ Connected | Order management system |
| `aws-luckyus-payment-rw` | ✅ Connected | Payment processing |
| `aws-luckyus-isalescrm-rw` | ✅ Connected | CRM data |
| `aws-luckyus-isalesmarket-rw` | ✅ Connected | Marketing data |
| `aws-luckyus-isalesmember-rw` | ✅ Connected | Member management |
| `aws-luckyus-scmshopstock-rw` | ✅ Connected | Inventory management |
| `aws-luckyus-scmasset-rw` | ✅ Connected | Asset management |
| `aws-luckyus-scmcommodity-rw` | ✅ Connected | Product catalog |
| `aws-luckyus-scmordering-rw` | ✅ Connected | Procurement orders |
| `aws-luckyus-ifiaccounting-rw` | ✅ Connected | Financial accounting |
| `aws-luckyus-ifichargecontrol-rw` | ✅ Connected | Charge control |
| `aws-luckyus-ifitax-rw` | ✅ Connected | Tax processing |
| `aws-luckyus-authservice-rw` | ✅ Connected | Authentication |
| `aws-luckyus-cmdb-rw` | ✅ Connected | Configuration management |
| `aws-luckyus-koala-rw` | ✅ Connected | Platform services |
| `aws-luckyus-ldas-rw` | ✅ Connected | Data analytics |
| `aws-luckyus-shop-rw` | ✅ Connected | Shop management |
| `aws-luckyus-shopexpand-rw` | ✅ Connected | Shop expansion |
| `aws-luckyus-empefficiency-rw` | ✅ Connected | Employee efficiency |
| ... | ... | *57 additional servers* |

**Connection Method**: MySQL protocol via MCP bridge
**Tool**: `mcp__mcp-db-gateway__mysql_query`

---

## 2. Redis Clusters (74 Total)

### MCP Server: `mcp-db-gateway`

| Cluster Name | Status | Primary Use |
|--------------|--------|-------------|
| `luckyus-apigateway` | ✅ Connected | API rate limiting, caching |
| `luckyus-auth` | ✅ Connected | Token cache, session storage |
| `luckyus-authservice` | ✅ Connected | Authentication tokens |
| `luckyus-unionauth` | ✅ Connected | Unified authentication |
| `luckyus-session` | ✅ Connected | User session storage |
| `luckyus-isales-order` | ✅ Connected | Order cache |
| `luckyus-isales-crm` | ✅ Connected | CRM cache |
| `luckyus-isales-market` | ✅ Connected | Marketing cache |
| `luckyus-isales-member` | ✅ Connected | Member data cache |
| `luckyus-isales-marketcapi` | ✅ Connected | Marketing API cache |
| `luckyus-scm-asset` | ✅ Connected | Asset cache |
| `luckyus-scm-commodity` | ✅ Connected | Product cache |
| `luckyus-scm-ordering` | ✅ Connected | Ordering cache |
| `luckyus-scm-shopstock` | ✅ Connected | Inventory cache |
| `luckyus-devops` | ✅ Connected | DevOps caching |
| `luckyus-cmdb` | ✅ Connected | CMDB cache |
| `luckyus-koala` | ✅ Connected | Platform cache |
| `luckyus-ldas` | ✅ Connected | Analytics cache |
| `luckyus-ifiaccounting` | ✅ Connected | Accounting cache |
| `luckyus-ifichargecontrol` | ✅ Connected | Charge control cache |
| `luckyus-ifitax` | ✅ Connected | Tax cache |
| `luckyus-redis-dify` | ✅ Connected | AI/LLM cache |
| `luckyus-bigdata-cyberdata` | ✅ Connected | Big data cache |
| `luckyus-bigdata-dataplatform` | ✅ Connected | Data platform cache |
| `luckyus-chronus` | ✅ Connected | Job scheduling |
| `luckyus-iehr` | ✅ Connected | HR system |
| `luckyus-empefficiency` | ✅ Connected | Efficiency tracking |
| `luckyus-ocp` | ✅ Connected | OCP services |
| `luckyus-shopexpand` | ✅ Connected | Shop expansion |
| ... | ... | *45 additional clusters* |

**Connection Method**: Redis protocol via AWS ElastiCache
**Tool**: `mcp__mcp-db-gateway__redis_command`

---

## 3. PostgreSQL Servers (3 Total)

### MCP Server: `mcp-db-gateway`

| Server Name | Status | Primary Use |
|-------------|--------|-------------|
| `aws-luckyus-dify-rw` | ✅ Connected | Dify AI platform |
| `aws-luckyus-difynew-rw` | ✅ Connected | New Dify instance |
| `aws-luckyus-pgilkmap-rw` | ✅ Connected | Map services |

**Connection Method**: PostgreSQL protocol via MCP bridge
**Tool**: `mcp__mcp-db-gateway__postgres_query`

---

## 4. Prometheus Datasources (3 Total)

### MCP Server: `grafana`

| Name | UID | Status | Primary Use |
|------|-----|--------|-------------|
| **UMBQuerier-Luckin** | `df8o21agxtkw0d` | ✅ Connected | PRIMARY - Node/EC2 metrics, K8s metrics |
| **prometheus** | `ff7hkeec6c9a8e` | ✅ Connected | General metrics |
| **prometheus_redis** | `ff6p0gjt24phce` | ✅ Connected | Redis cluster metrics |

### Prometheus Scrape Pools (Sample)

| Scrape Pool | Targets | Status |
|-------------|---------|--------|
| `aws-redis-job` | 74 | ✅ Healthy |
| `node-exporter` | 100+ | ✅ Healthy |
| `aws-node-exporter` | 100+ | ✅ Healthy |
| `kube-state-metrics` | 5 | ✅ Healthy |
| `db-*` jobs | 60+ | ✅ Healthy |

**Connection Method**: PromQL via Grafana proxy
**Tool**: `mcp__grafana__query_prometheus`

---

## 5. CloudWatch (AWS)

### MCP Server: `cloudwatch-server`

#### Log Groups (64+ RDS Slow Query Logs)

| Log Group Pattern | Count | Status | Primary Use |
|-------------------|-------|--------|-------------|
| `/aws/rds/cluster/*/slowquery` | 52 | ✅ Connected | RDS slow query analysis |
| `/aws/rds/instance/*/slowquery` | 12 | ✅ Connected | RDS instance slow queries |
| `/aws/eks/*/logs` | Multiple | ✅ Connected | EKS application logs |
| `/aws/lambda/*` | Multiple | ✅ Connected | Lambda function logs |

#### Metric Namespaces

| Namespace | Status | Primary Use |
|-----------|--------|-------------|
| `AWS/EC2` | ✅ Connected | EC2 instance metrics |
| `AWS/RDS` | ✅ Connected | RDS database metrics |
| `AWS/ElastiCache` | ✅ Connected | Redis/ElastiCache metrics |
| `AWS/ES` | ✅ Connected | OpenSearch/ES metrics |
| `AWS/EKS` | ✅ Connected | EKS cluster metrics |
| `ContainerInsights` | ✅ Connected | Container metrics |

**Connection Method**: AWS SDK via MCP server
**Tools**:
- `mcp__cloudwatch-server__get_metric_data`
- `mcp__cloudwatch-server__execute_log_insights_query`
- `mcp__cloudwatch-server__get_active_alarms`

---

## 6. EKS Clusters

### MCP Server: `eks-server`

| Cluster Name | Status | Primary Use |
|--------------|--------|-------------|
| `prod-worker01-eks-us` | ✅ Connected | Production workloads |

#### Kubernetes Resources Accessible

| Resource Type | API Version | Status |
|---------------|-------------|--------|
| Pods | v1 | ✅ Available |
| Deployments | apps/v1 | ✅ Available |
| Services | v1 | ✅ Available |
| Events | v1 | ✅ Available |
| Nodes | v1 | ✅ Available |
| ConfigMaps | v1 | ✅ Available |
| Secrets | v1 | ⚠️ Requires `--allow-sensitive-data-access` |

**Connection Method**: Kubernetes API via EKS
**Tools**:
- `mcp__eks-server__list_k8s_resources`
- `mcp__eks-server__get_pod_logs`
- `mcp__eks-server__get_k8s_events`

---

## 7. Grafana Services

### MCP Server: `grafana`

| Service | Status | Primary Use |
|---------|--------|-------------|
| Dashboards | ✅ Connected | Dashboard management |
| Alerting | ✅ Connected | Alert rule management |
| Datasources | ✅ Connected | Data source queries |

#### Grafana Datasources

| Name | Type | UID | Status |
|------|------|-----|--------|
| UMBQuerier-Luckin | prometheus | `df8o21agxtkw0d` | ✅ Default |
| prometheus | prometheus | `ff7hkeec6c9a8e` | ✅ Active |
| prometheus_redis | prometheus | `ff6p0gjt24phce` | ✅ Active |
| elasticsearch | elasticsearch | `cdlcdlcmb58g0e` | ✅ Active |
| mysql-devops | mysql | (varies) | ✅ Active |
| mysql-cmdb | mysql | (varies) | ✅ Active |
| mysql-salesorder | mysql | (varies) | ✅ Active |

**Connection Method**: Grafana HTTP API
**Tools**: Multiple `mcp__grafana__*` tools

---

## 8. Elasticsearch/OpenSearch

### Access via CloudWatch

| Domain Pattern | Status | Primary Use |
|----------------|--------|-------------|
| `luckyus-*-es` | ✅ Connected | Log aggregation, search |

**Connection Method**: CloudWatch metrics (`AWS/ES` namespace)
**Tool**: `mcp__cloudwatch-server__get_metric_data`

---

## Connection Verification Commands

### Test MySQL Connection
```sql
-- Server: aws-luckyus-devops-rw
SELECT 1 as connection_test;
```

### Test Redis Connection
```
-- Server: luckyus-apigateway
PING
```

### Test Prometheus Connection
```promql
-- Datasource UID: df8o21agxtkw0d
up
```

### Test CloudWatch Connection
```
-- Namespace: AWS/EC2
-- Metric: CPUUtilization
```

---

## Summary by MCP Server

| MCP Server | Data Sources | Connection Type |
|------------|--------------|-----------------|
| `mcp-db-gateway` | 61 MySQL, 74 Redis, 3 PostgreSQL | Database protocols |
| `grafana` | 3 Prometheus, 1 ES, Dashboards | HTTP API |
| `cloudwatch-server` | AWS Metrics, Logs, Alarms | AWS SDK |
| `eks-server` | 1 EKS cluster, K8s resources | Kubernetes API |
| `prometheus` | Direct Prometheus access | PromQL |

---

*Document Version: 1.0 | Last Updated: 2026-01-18*
