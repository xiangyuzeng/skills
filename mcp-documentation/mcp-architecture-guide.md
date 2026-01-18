# MCP Architecture Guide

> Version: 1.0 | Last Updated: 2026-01-18

## 1. Overview

### What is MCP?

**Model Context Protocol (MCP)** is an open protocol developed by Anthropic that enables AI assistants like Claude to securely connect to external data sources and tools. MCP provides a standardized way for Claude Code to access databases, APIs, monitoring systems, and cloud services without requiring direct credential exposure to the AI model.

### How MCP Works with Claude Code

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Claude Code Session                                │
│  ┌─────────────────┐                                                        │
│  │   User Query    │──▶ "Investigate this Redis alert..."                   │
│  └────────┬────────┘                                                        │
│           │                                                                  │
│           ▼                                                                  │
│  ┌─────────────────┐      ┌──────────────────┐      ┌───────────────────┐  │
│  │  Claude (Opus)  │──────│  MCP Protocol    │──────│  MCP Servers      │  │
│  │  AI Reasoning   │◀─────│  JSON-RPC 2.0    │◀─────│  (Docker/Local)   │  │
│  └─────────────────┘      └──────────────────┘      └───────────────────┘  │
│                                                              │               │
└──────────────────────────────────────────────────────────────│───────────────┘
                                                               │
                                    ┌──────────────────────────┴───────────────┐
                                    │           Backend Services               │
                                    │  ┌─────────┐ ┌─────────┐ ┌─────────┐    │
                                    │  │   RDS   │ │ Redis   │ │  EKS    │    │
                                    │  │ (MySQL) │ │ Elastic │ │ K8s API │    │
                                    │  └─────────┘ └─────────┘ └─────────┘    │
                                    │  ┌─────────┐ ┌─────────┐ ┌─────────┐    │
                                    │  │Grafana  │ │CloudWatch││Prometheus│   │
                                    │  │   API   │ │   API   │ │   API   │    │
                                    │  └─────────┘ └─────────┘ └─────────┘    │
                                    └──────────────────────────────────────────┘
```

### Key Benefits

| Benefit | Description |
|---------|-------------|
| **Security** | Credentials never exposed to AI model |
| **Standardization** | Consistent protocol for all data sources |
| **Extensibility** | Easy to add new MCP servers |
| **Isolation** | Each server runs in isolated container |
| **Auditability** | All queries logged and traceable |

---

## 2. Components

### 2.1 Claude Code Client

The Claude Code client is the primary interface for users to interact with Claude. It:

- Accepts natural language queries and commands
- Parses skill invocations (e.g., `/investigate-ec2`)
- Coordinates MCP tool calls
- Renders results in markdown format

```
┌────────────────────────────────────────────┐
│              Claude Code CLI               │
├────────────────────────────────────────────┤
│  • Terminal-based interface                │
│  • Conversation context management         │
│  • Tool orchestration                      │
│  • Response formatting                     │
└────────────────────────────────────────────┘
```

### 2.2 MCP Protocol Layer

MCP uses **JSON-RPC 2.0** as its transport protocol. Each MCP server exposes a set of **tools** that Claude can invoke.

#### Protocol Specification

```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "mcp__grafana__query_prometheus",
    "arguments": {
      "datasourceUid": "df8o21agxtkw0d",
      "expr": "up{job='node-exporter'}",
      "startTime": "now-1h",
      "queryType": "instant"
    }
  },
  "id": 1
}
```

#### Tool Naming Convention

```
mcp__<server_name>__<tool_name>
     │              │
     │              └─── Specific operation (query_prometheus, mysql_query)
     └────────────────── MCP server identifier (grafana, mcp-db-gateway)
```

### 2.3 Docker MCP Bridges

MCP servers run as containerized services, providing isolation and easy deployment.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Docker MCP Bridge Layer                             │
│                                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │  mcp-db-gateway │  │  grafana-mcp    │  │ cloudwatch-mcp  │              │
│  │                 │  │                 │  │                 │              │
│  │ • MySQL (61)    │  │ • Prometheus(3) │  │ • Metrics       │              │
│  │ • Redis (74)    │  │ • Dashboards    │  │ • Logs          │              │
│  │ • PostgreSQL(3) │  │ • Alerting      │  │ • Alarms        │              │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘              │
│           │                    │                    │                        │
│           ▼                    ▼                    ▼                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │  eks-mcp        │  │  prometheus-mcp │  │  pricing-mcp    │              │
│  │                 │  │                 │  │                 │              │
│  │ • K8s Resources │  │ • Direct PromQL │  │ • AWS Pricing   │              │
│  │ • Pod Logs      │  │ • Targets       │  │ • Cost Analysis │              │
│  │ • Events        │  │ • Metadata      │  │                 │              │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘              │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

#### MCP Server Configuration

```yaml
# Example MCP Server Configuration
mcpServers:
  mcp-db-gateway:
    image: mcp-db-gateway:latest
    environment:
      - MYSQL_SERVERS_CONFIG=/config/mysql-servers.json
      - REDIS_SERVERS_CONFIG=/config/redis-servers.json
      - POSTGRES_SERVERS_CONFIG=/config/postgres-servers.json
    volumes:
      - ./config:/config:ro

  grafana:
    image: grafana-mcp:latest
    environment:
      - GRAFANA_URL=https://grafana.internal.com
      - GRAFANA_API_KEY_FILE=/secrets/grafana-api-key

  cloudwatch-server:
    image: cloudwatch-mcp:latest
    environment:
      - AWS_REGION=us-east-1
      - AWS_PROFILE=production
```

### 2.4 Backend Services

The actual data sources that MCP servers connect to:

| Service Category | Services | Protocol |
|------------------|----------|----------|
| **Databases** | AWS RDS (MySQL), ElastiCache (Redis), RDS (PostgreSQL) | Native DB protocols |
| **Monitoring** | Prometheus, Grafana, CloudWatch | HTTP/HTTPS APIs |
| **Orchestration** | Amazon EKS, Kubernetes | K8s API |
| **Search** | OpenSearch/Elasticsearch | REST API |
| **AWS Services** | Cost Explorer, IAM, EC2 | AWS SDK |

---

## 3. Connection Flow

### Step-by-Step: Query to Response

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                         CONNECTION FLOW DIAGRAM                               │
│                                                                               │
│  STEP 1: User Query                                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │  User: "Investigate this Redis alert: luckyus-isales-order memory 75%"  │  │
│  └────────────────────────────────────────────┬────────────────────────────┘  │
│                                               │                               │
│  STEP 2: Claude Reasoning                     ▼                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │  Claude identifies:                                                      │  │
│  │  • Alert type: Redis memory                                             │  │
│  │  • Cluster: luckyus-isales-order                                        │  │
│  │  • Required tools: grafana (prometheus), mcp-db-gateway (redis)         │  │
│  └────────────────────────────────────────────┬────────────────────────────┘  │
│                                               │                               │
│  STEP 3: MCP Tool Invocation                  ▼                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │  Claude invokes multiple tools in parallel:                             │  │
│  │                                                                          │  │
│  │  Tool 1: mcp__grafana__query_prometheus                                 │  │
│  │  ├─ datasourceUid: ff6p0gjt24phce (prometheus_redis)                    │  │
│  │  └─ expr: redis_memory_used_bytes{instance=~".*isales-order.*"}         │  │
│  │                                                                          │  │
│  │  Tool 2: mcp__mcp-db-gateway__redis_command                             │  │
│  │  ├─ server: luckyus-isales-order                                        │  │
│  │  └─ command: INFO memory                                                │  │
│  │                                                                          │  │
│  │  Tool 3: mcp__cloudwatch-server__get_metric_data                        │  │
│  │  ├─ namespace: AWS/ElastiCache                                          │  │
│  │  └─ metricName: DatabaseMemoryUsagePercentage                           │  │
│  └────────────────────────────────────────────┬────────────────────────────┘  │
│                                               │                               │
│  STEP 4: MCP Protocol Routing                 ▼                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │  MCP Router sends JSON-RPC requests to appropriate servers:             │  │
│  │                                                                          │  │
│  │  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐             │  │
│  │  │   Grafana    │     │ DB Gateway   │     │  CloudWatch  │             │  │
│  │  │   Server     │     │   Server     │     │   Server     │             │  │
│  │  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘             │  │
│  └─────────│────────────────────│────────────────────│─────────────────────┘  │
│            │                    │                    │                        │
│  STEP 5: Backend Queries        ▼                    ▼                        │
│  ┌─────────┴────────────────────┴────────────────────┴─────────────────────┐  │
│  │  Each MCP server queries its backend:                                   │  │
│  │                                                                          │  │
│  │  Grafana → Prometheus API → PromQL query → Time series data             │  │
│  │  DB Gateway → ElastiCache → Redis RESP protocol → INFO output           │  │
│  │  CloudWatch → AWS API → GetMetricData → Metric values                   │  │
│  └────────────────────────────────────────────┬────────────────────────────┘  │
│                                               │                               │
│  STEP 6: Response Aggregation                 ▼                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │  Responses flow back through MCP:                                       │  │
│  │                                                                          │  │
│  │  Tool 1 Result: { memory_used: 1.5GB, memory_max: 2GB, usage: 75% }    │  │
│  │  Tool 2 Result: { used_memory: 1610612736, maxmemory: 2147483648, ... } │  │
│  │  Tool 3 Result: { datapoints: [...], average: 74.8% }                   │  │
│  └────────────────────────────────────────────┬────────────────────────────┘  │
│                                               │                               │
│  STEP 7: Claude Analysis & Report             ▼                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │  Claude synthesizes results into investigation report:                  │  │
│  │                                                                          │  │
│  │  ## Redis Investigation Report                                          │  │
│  │  | Metric | Value | Status |                                            │  │
│  │  | Memory Usage | 75% | ⚠️ WARNING |                                    │  │
│  │  | Evictions | 0 | ✅ OK |                                               │  │
│  │  ...                                                                     │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

### Timing Breakdown

| Step | Duration | Notes |
|------|----------|-------|
| User Query Parsing | ~100ms | Intent extraction |
| Tool Selection | ~200ms | Claude reasoning |
| MCP Routing | ~50ms | Protocol overhead |
| Backend Query | 100-2000ms | Varies by source |
| Response Processing | ~100ms | JSON parsing |
| Report Generation | ~500ms | Claude synthesis |
| **Total** | **1-3 seconds** | Typical end-to-end |

---

## 4. Security Model

### 4.1 Credential Isolation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SECURITY BOUNDARY MODEL                              │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     Claude Code Session                               │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  Claude AI Model                                                 │  │  │
│  │  │  • NO access to credentials                                      │  │  │
│  │  │  • NO access to connection strings                               │  │  │
│  │  │  • Only sees tool results (sanitized)                            │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                              MCP Protocol                                   │
│                          (No credentials in payload)                        │
│                                    │                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     MCP Server Containers                             │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  Credentials stored in:                                          │  │  │
│  │  │  • Environment variables                                         │  │  │
│  │  │  • Mounted secret files                                          │  │  │
│  │  │  • AWS IAM roles                                                 │  │  │
│  │  │  • Kubernetes secrets                                            │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Security Layers

| Layer | Protection | Implementation |
|-------|------------|----------------|
| **Network** | TLS/mTLS | All MCP connections encrypted |
| **Authentication** | IAM/API Keys | Each server has own credentials |
| **Authorization** | Read-only by default | Write operations require explicit flags |
| **Isolation** | Container sandboxing | Each MCP server in own container |
| **Audit** | Query logging | All tool calls logged with timestamps |

### 4.3 Access Control Flags

```bash
# MCP Server Security Flags

# Read-only mode (default)
--read-only

# Enable write operations (dangerous)
--allow-write

# Enable sensitive data access (secrets, credentials)
--allow-sensitive-data-access

# Enable security scanning before writes
--security-scanning=enabled
```

### 4.4 Credential Flow

```
┌────────────────────────────────────────────────────────────────────────────┐
│                       CREDENTIAL NEVER LEAVES                              │
│                                                                            │
│   ┌─────────────┐                              ┌─────────────────────────┐ │
│   │   Claude    │                              │    MCP DB Gateway       │ │
│   │             │  Tool Call:                  │                         │ │
│   │  "Query DB  │  ─────────────────────────▶  │  ┌─────────────────┐   │ │
│   │   for user  │  { server: "devops-rw",      │  │  Credential     │   │ │
│   │   data"     │    sql: "SELECT..." }        │  │  Store          │   │ │
│   │             │                              │  │                 │   │ │
│   │  No access  │                              │  │  devops-rw:     │   │ │
│   │  to creds   │  Result:                     │  │  - host: ****   │   │ │
│   │             │  ◀─────────────────────────  │  │  - user: ****   │   │ │
│   │             │  { rows: [...] }             │  │  - pass: ****   │   │ │
│   └─────────────┘  (data only, no creds)       │  └─────────────────┘   │ │
│                                                └─────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Data Endpoint Summary

### 5.1 Total Counts by Type

| Category | Type | Count | MCP Server |
|----------|------|-------|------------|
| **Databases** | MySQL | 61 | mcp-db-gateway |
| | Redis | 74 | mcp-db-gateway |
| | PostgreSQL | 3 | mcp-db-gateway |
| **Monitoring** | Prometheus Datasources | 3 | grafana |
| | Prometheus Targets | 200+ | prometheus |
| | CloudWatch Log Groups | 64+ | cloudwatch-server |
| | CloudWatch Metrics | Unlimited | cloudwatch-server |
| **Orchestration** | EKS Clusters | 1 | eks-server |
| | K8s Namespaces | 40+ | eks-server |
| **Search** | Elasticsearch | 1 | grafana |
| **Total** | **All Sources** | **200+** | |

### 5.2 Endpoint Distribution

```
                    MCP Data Source Distribution

    ┌────────────────────────────────────────────────────────┐
    │ Redis Clusters                                         │
    │ ████████████████████████████████████████████ 74 (37%) │
    ├────────────────────────────────────────────────────────┤
    │ MySQL Servers                                          │
    │ ██████████████████████████████████ 61 (30%)           │
    ├────────────────────────────────────────────────────────┤
    │ CloudWatch Log Groups                                  │
    │ ██████████████████████████████████ 64 (32%)           │
    ├────────────────────────────────────────────────────────┤
    │ Other (Prometheus, ES, EKS, PostgreSQL)                │
    │ ████ 8 (4%)                                           │
    └────────────────────────────────────────────────────────┘
```

### 5.3 Tools by MCP Server

#### mcp-db-gateway (138 endpoints)
```
Tools:
├── mcp__mcp-db-gateway__list_servers
├── mcp__mcp-db-gateway__mysql_query
├── mcp__mcp-db-gateway__redis_command
└── mcp__mcp-db-gateway__postgres_query
```

#### grafana (30+ tools)
```
Tools:
├── mcp__grafana__query_prometheus
├── mcp__grafana__list_prometheus_label_names
├── mcp__grafana__list_prometheus_label_values
├── mcp__grafana__list_datasources
├── mcp__grafana__search_dashboards
├── mcp__grafana__get_dashboard_by_uid
├── mcp__grafana__list_alert_rules
└── ... (20+ more)
```

#### cloudwatch-server (10+ tools)
```
Tools:
├── mcp__cloudwatch-server__describe_log_groups
├── mcp__cloudwatch-server__execute_log_insights_query
├── mcp__cloudwatch-server__get_metric_data
├── mcp__cloudwatch-server__get_active_alarms
├── mcp__cloudwatch-server__get_alarm_history
├── mcp__cloudwatch-server__analyze_log_group
└── ... (5+ more)
```

#### eks-server (15+ tools)
```
Tools:
├── mcp__eks-server__list_k8s_resources
├── mcp__eks-server__get_pod_logs
├── mcp__eks-server__get_k8s_events
├── mcp__eks-server__manage_k8s_resource
├── mcp__eks-server__apply_yaml
├── mcp__eks-server__get_cloudwatch_logs
└── ... (10+ more)
```

#### prometheus (10 tools)
```
Tools:
├── mcp__prometheus__prometheus_query
├── mcp__prometheus__prometheus_query_range
├── mcp__prometheus__prometheus_list_metrics
├── mcp__prometheus__prometheus_metric_metadata
├── mcp__prometheus__prometheus_list_labels
├── mcp__prometheus__prometheus_label_values
└── mcp__prometheus__prometheus_list_targets
```

---

## 6. Quick Reference

### Common Tool Patterns

```python
# Query MySQL
mcp__mcp-db-gateway__mysql_query(
    server="aws-luckyus-devops-rw",
    sql="SELECT * FROM service_registry LIMIT 10"
)

# Query Redis
mcp__mcp-db-gateway__redis_command(
    server="luckyus-apigateway",
    command="INFO",
    args=["memory"]
)

# Query Prometheus
mcp__grafana__query_prometheus(
    datasourceUid="df8o21agxtkw0d",
    expr="up{job='node-exporter'}",
    startTime="now-1h",
    queryType="instant"
)

# Query CloudWatch Logs
mcp__cloudwatch-server__execute_log_insights_query(
    log_group_names=["/aws/rds/cluster/devops/slowquery"],
    query_string="fields @timestamp, @message | limit 50",
    start_time="2026-01-18T00:00:00Z",
    end_time="2026-01-18T12:00:00Z"
)

# Get K8s Resources
mcp__eks-server__list_k8s_resources(
    cluster_name="prod-worker01-eks-us",
    kind="Pod",
    api_version="v1",
    namespace="production"
)
```

### Datasource UIDs (Prometheus)

| Name | UID | Use |
|------|-----|-----|
| UMBQuerier-Luckin | `df8o21agxtkw0d` | Primary - Node/EC2/K8s |
| prometheus | `ff7hkeec6c9a8e` | General metrics |
| prometheus_redis | `ff6p0gjt24phce` | Redis metrics |

### Key Database Servers

| Server | Purpose |
|--------|---------|
| `aws-luckyus-devops-rw` | Service registry, alert logs |
| `aws-luckyus-salesorder-rw` | Order processing |
| `aws-luckyus-payment-rw` | Payment processing |

---

*Document Version: 1.0 | Last Updated: 2026-01-18*
