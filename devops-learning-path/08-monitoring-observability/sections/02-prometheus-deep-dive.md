# Module 08: Monitoring & Observability
## Section 02: Prometheus Deep Dive - Architecture, PromQL, and Advanced Metrics

---

## Table of Contents

1. [Introduction](#introduction)
2. [Prometheus Architecture Deep Dive](#prometheus-architecture-deep-dive)
3. [Data Model Fundamentals](#data-model-fundamentals)
4. [PromQL: The Query Language](#promql-the-query-language)
5. [Recording Rules & Pre-computation](#recording-rules--pre-computation)
6. [Service Discovery Mechanisms](#service-discovery-mechanisms)
7. [Advanced Alerting with Prometheus](#advanced-alerting-with-prometheus)
8. [Hands-on Lab: Complete Prometheus Setup](#hands-on-lab-complete-prometheus-setup)
9. [Best Practices & Common Pitfalls](#best-practices--common-pitfalls)
10. [Troubleshooting Guide](#troubleshooting-guide)

---

## Introduction

Prometheus has become the de facto standard for monitoring cloud-native applications and Kubernetes clusters. This deep dive will take you from basic understanding to advanced mastery of Prometheus, covering architecture decisions, query optimization, and production-ready configurations.

### Why Prometheus?

- **Pull-based model** for better control over scraping
- **Multi-dimensional data model** with labels
- **Powerful query language** (PromQL)
- **No dependencies on distributed storage**
- **Time series data stored efficiently**
- **Alertmanager integration** for sophisticated alerting
- **Huge ecosystem** of exporters and integrations

---

## Prometheus Architecture Deep Dive

### Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Prometheus Server                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │   Retrieval  │  │     TSDB     │  │    HTTP Server   │   │
│  │   (Scraper)  │→ │   (Storage)  │← │   (Query API)    │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
│         ↑                                       ↓           │
│         │                                       │           │
│  ┌──────────────┐                       ┌──────────────┐   │
│  │   Service    │                       │  Alertmanager│   │
│  │  Discovery   │                       │   (External) │   │
│  └──────────────┘                       └──────────────┘   │
└─────────────────────────────────────────────────────────────┘
         ↑
         │
    ┌────┴────┐
    │Targets  │
    │(Exporters)│
    └─────────┘
```

### 1. Retrieval Layer (Scraper)

The retrieval layer is responsible for pulling metrics from configured targets at regular intervals.

**Key Configuration Parameters:**

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    scrape_interval: 15s        # How often to scrape
    scrape_timeout: 10s         # Timeout for each scrape
    honor_labels: true          # Preserve target labels
    honor_timestamps: true      # Use target timestamps
    metrics_path: /metrics      # Endpoint path
    scheme: http                # http or https
    follow_redirects: true      # Follow HTTP redirects
    
    static_configs:
      - targets: ['localhost:9100']
        labels:
          env: 'production'
          team: 'infrastructure'
    
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '(.+):(\d+)'
        replacement: '${1}'
```

**Scrape Interval Strategy:**

| Use Case | Recommended Interval | Rationale |
|----------|---------------------|-----------|
| Infrastructure metrics | 15-30s | Balance between detail and overhead |
| Application metrics | 5-15s | Faster detection of issues |
| Business metrics | 1-5min | Lower frequency acceptable |
| High-cardinality metrics | 30-60s | Reduce storage pressure |

### 2. Time Series Database (TSDB)

Prometheus uses a custom TSDB optimized for time-series data with label-based indexing.

**TSDB Structure:**

```
prometheus-data/
├── wal/              # Write-Ahead Log (recent data)
├── chunks_head/      # In-memory head chunk
└── blocks/           # Compacted blocks
    ├── 01BKGV7JBM69T2G1BGBGM6KB12/
    │   ├── meta.json
    │   ├── index
    │   ├── chunks/
    │   └── tombstones
    └── ...
```

**Storage Configuration:**

```yaml
# prometheus.yml
storage:
  tsdb:
    retention.time: 15d          # Time-based retention
    retention.size: 50GB         # Size-based retention
    max-block-duration: 2h       # Block duration
    min-block-duration: 2h       # Minimum block duration
    no-lockfile: false           # Lock file for safety
    wal-compression: true        # Compress WAL
```

**Performance Optimization:**

```yaml
# Memory tuning
storage.tsdb.wal-segment-size: 64MB
storage.tsdb.head-chunks-write-queue-size: 1000

# Compaction tuning
storage.tsdb.min-block-duration: 2h
storage.tsdb.max-block-duration: 2h
storage.tsdb.no-lockfile: false
```

### 3. HTTP Server & Query API

Prometheus exposes multiple endpoints for querying and management:

| Endpoint | Purpose | Example |
|----------|---------|---------|
| `/api/v1/query` | Instant query | `query=up{job="node"}` |
| `/api/v1/query_range` | Range query | `start=...&end=...&step=...` |
| `/api/v1/series` | Get series metadata | `match[]=up{job="node"}` |
| `/api/v1/labels` | Get all label names | - |
| `/api/v1/label/<name>/values` | Get label values | - |
| `/api/v1/targets` | Get scrape targets | - |
| `/api/v1/rules` | Get alert/recording rules | - |
| `/api/v1/alerts` | Get active alerts | - |

**Query Performance Tips:**

```bash
# Use specific label matchers
node_cpu_seconds_total{mode="idle",instance="server1"}

# Avoid regex when possible (slower)
http_requests_total{job=~"api.*"}  # Slower
http_requests_total{job="api-server"}  # Faster

# Use vector selectors efficiently
rate(http_requests_total[5m])  # Good
rate(http_requests_total[1h])  # Expensive
```

---

## Data Model Fundamentals

### Metric Types

Prometheus supports four core metric types:

#### 1. Counter

Monotonically increasing value that only goes up or resets to zero.

```promql
# Examples
http_requests_total
node_cpu_seconds_total
process_cpu_seconds_total

# Query patterns
# Rate over time window
rate(http_requests_total[5m])

# Increase over time window
increase(http_requests_total[1h])

# Predict future value based on trend
predict_linear(http_requests_total[1h], 3600)
```

#### 2. Gauge

Value that can go up or down arbitrarily.

```promql
# Examples
node_memory_MemAvailable_bytes
process_resident_memory_bytes
temperature_degrees

# Query patterns
# Current value
node_memory_MemAvailable_bytes

# Min/Max over time
min_over_time(node_memory_MemAvailable_bytes[1h])
max_over_time(node_memory_MemAvailable_bytes[1h])

# Changes in value
changes(node_memory_MemAvailable_bytes[1h])

# Derivative (rate of change)
deriv(node_memory_MemAvailable_bytes[5m])
```

#### 3. Histogram

Samples observations into configurable buckets.

```promql
# Examples
http_request_duration_seconds_bucket
apiserver_request_duration_seconds_bucket

# Query patterns
# Count of observations in bucket
http_request_duration_seconds_bucket{le="0.5"}

# Total count of observations
http_request_duration_seconds_count

# Sum of all observed values
http_request_duration_seconds_sum

# Calculate percentile (e.g., p95)
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket[5m]))

# Average observation value
http_request_duration_seconds_sum / http_request_duration_seconds_count
```

#### 4. Summary

Similar to histogram but calculates quantiles on the client side.

```promql
# Examples
grpc_client_handling_seconds
go_gc_duration_seconds

# Query patterns
# Access quantile directly
grpc_client_handling_seconds{quantile="0.95"}

# Count and sum
grpc_client_handling_seconds_count
grpc_client_handling_seconds_sum
```

### Labels and Dimensions

Labels are key-value pairs that provide dimensions for metrics:

```promql
# Basic label matching
http_requests_total{method="POST",status="200"}

# Negative matching
http_requests_total{status!="500"}

# Regex matching
http_requests_total{job=~"api-.*"}

# Multiple label matchers
http_requests_total{
  method="POST",
  status=~"2..",
  handler=~"/api/v[12]/.*"
}
```

**Label Best Practices:**

✅ **DO:**
- Use lowercase letters and underscores
- Keep cardinality low (< 100k unique combinations)
- Use meaningful, consistent naming
- Include environment, service, instance labels

❌ **DON'T:**
- Use high-cardinality labels (user IDs, URLs, IPs)
- Change label sets dynamically
- Use empty label values
- Create labels with unbounded values

**High Cardinality Example (BAD):**

```promql
# DON'T - Creates millions of series
http_requests_total{user_id="12345", url="/api/users/12345"}

# DO - Aggregate appropriately
http_requests_total{endpoint="/api/users", method="GET"}
```

---

## PromQL: The Query Language

### Basic Syntax

#### Vector Selectors

```promql
# Simple selector
http_requests_total

# With label matchers
http_requests_total{job="api-server", method="POST"}

# Match all jobs starting with 'api'
http_requests_total{job=~"api-.*"}

# Exclude specific instances
http_requests_total{instance!="bad-server:8080"}
```

#### Range Vectors

```promql
# 5-minute rate
rate(http_requests_total[5m])

# 1-hour increase
increase(http_requests_total[1h])

# Average over 10 minutes
avg_over_node_cpu_seconds_total[10m])

# Maximum in last hour
max_over_time(node_memory_Used_bytes[1h])
```

### Essential Functions

#### Rate Functions

```promql
# Per-second average rate
rate(http_requests_total[5m])

# Per-second instantaneous rate
irate(http_requests_total[5m])

# Rate of increase (counters only)
increase(http_requests_total[1h])

# Predict linear trend
predict_linear(http_requests_total[1h], 3600)
```

**Rate vs Irate:**
- `rate()`: Smoothed average, better for alerting
- `irate()`: More volatile, better for graphs

#### Aggregation Operators

```promql
# Sum across all instances
sum(http_requests_total)

# Average by job
avg by (job) (http_requests_total)

# Count instances per environment
count by (env) (up)

# Maximum memory usage per node
max by (instance) (node_memory_Used_bytes)

# Top 5 CPU consumers
topk(5, rate(node_cpu_seconds_total{mode="user"}[5m]))

# Bottom 3 by available memory
bottomk(3, node_memory_MemAvailable_bytes)
```

**Aggregation Modifiers:**

```promql
# Group by specific labels
sum by (job, instance) (http_requests_total)

# Ignore specific labels
sum without (pod, container) (container_memory_usage_bytes)

# Group left/right joins
sum by (instance) (
  node_filesystem_size_bytes 
  * on (device, instance) group_left(mountpoint)
  node_filesystem_avail_bytes
)
```

#### Mathematical Operations

```promql
# Basic arithmetic
node_memory_Total_bytes - node_memory_MemAvailable_bytes

# Percentage calculation
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Ratio
http_requests_total{status="500"} / http_requests_total

# Boolean comparisons
up == 1
node_memory_MemAvailable_bytes < 100000000
```

#### Time Functions

```promql
# Current Unix timestamp
time()

# Days since epoch
days_in_month()
day_of_month()
day_of_week()
hour()
minute()
month()
year()
```

### Advanced Query Patterns

#### Error Rate Calculation

```promql
# Error rate as percentage
sum(rate(http_requests_total{status=~"5.."}[5m])) 
/ 
sum(rate(http_requests_total[5m])) 
* 100

# Errors per second
sum(rate(http_requests_total{status=~"4..|5.."}[5m]))
```

#### Latency Percentiles

```promql
# 95th percentile response time
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# 99th percentile by endpoint
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{endpoint=~"/api/.*"}[5m])) 
  by (le, endpoint)
)
```

#### Availability Calculation

```promql
# Service availability percentage
avg_over_time(up{job="api-server"}[24h]) * 100

# Uptime in hours
sum_over_time(up{job="api-server"}[24h]) / (24 * 60)
```

#### Resource Utilization

```promql
# CPU utilization percentage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory utilization percentage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk utilization percentage
(1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100
```

#### Anomaly Detection

```promql
# Standard deviation from mean
(abs(
  node_load1 - avg_over_time(node_load1[24h])
) / stddev_over_time(node_load1[24h])) > 3

# Sudden spike detection
rate(http_requests_total[1m]) > 
(avg_over_time(rate(http_requests_total[1h])[1h]) * 2)
```

---

## Recording Rules & Pre-computation

### What are Recording Rules?

Recording rules pre-compute frequently used or computationally expensive expressions and save the result as a new time series.

### Benefits

- **Performance**: Faster query execution
- **Simplification**: Complex queries become simple
- **Consistency**: Same calculation everywhere
- **Cost Reduction**: Less computation on dashboards

### Creating Recording Rules

```yaml
# prometheus-rules.yml
groups:
  - name: recording_rules
    interval: 30s
    rules:
      # Pre-compute request rate
      - record: job:http_requests_total:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
      
      # Pre-compute error rate
      - record: job:http_errors:ratio_rate5m
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum by (job) (rate(http_requests_total[5m]))
      
      # Pre-compute latency percentiles
      - record: job:http_request_duration_seconds:p95_5m
        expr: |
          histogram_quantile(0.95,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
      
      # Pre-compute CPU utilization
      - record: instance:node_cpu_utilization:ratio_rate5m
        expr: |
          1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
      
      # Pre-compute memory utilization
      - record: instance:node_memory_utilization:ratio
        expr: |
          1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```

### Naming Conventions

Follow this pattern: `level:metric:operations`

```yaml
# Level indicators
job:metric:operation      # Aggregated by job
instance:metric:operation # Aggregated by instance
cluster:metric:operation  # Aggregated by cluster

# Operation indicators
rate5m                  # 5-minute rate
ratio                   # Ratio/percentage
p95                     # 95th percentile
avg                     # Average
sum                     # Sum
```

### Best Practices

✅ **DO:**
- Use recording rules for complex, frequently-used queries
- Document the purpose of each rule
- Test rules before deploying
- Monitor rule evaluation times

❌ **DON'T:**
- Create rules for simple queries
- Duplicate existing metrics
- Use recording rules for one-off analysis
- Forget to update dashboards after creating rules

### Rule Evaluation Monitoring

```promql
# Check rule evaluation duration
prometheus_rule_evaluation_duration_seconds

# Check for failed evaluations
prometheus_rule_evaluation_failures_total

# Rule group evaluation metrics
prometheus_rule_group_last_evaluation_timestamp_seconds
prometheus_rule_group_last_duration_seconds
prometheus_rule_group_iterations_total
```

---

## Service Discovery Mechanisms

### Static Configuration

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: 
          - 'server1:9100'
          - 'server2:9100'
          - 'server3:9100'
        labels:
          env: 'production'
          datacenter: 'us-east-1'
```

### File-Based Service Discovery

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/node-*.json'
        refresh_interval: 30s
```

**Target File Format (JSON):**

```json
[
  {
    "targets": ["server1:9100", "server2:9100"],
    "labels": {
      "env": "production",
      "team": "infrastructure"
    }
  },
  {
    "targets": ["server3:9100"],
    "labels": {
      "env": "staging",
      "team": "development"
    }
  }
]
```

**Target File Format (YAML):**

```yaml
- targets:
    - server1:9100
    - server2:9100
  labels:
    env: production
    team: infrastructure

- targets:
    - server3:9100
  labels:
    env: staging
    team: development
```

### Kubernetes Service Discovery

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
        servers:
          - https://kubernetes.default.svc
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    
    relabel_configs:
      # Only scrape pods with annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      
      # Use custom port if specified
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: ${1}
      
      # Add pod name as label
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      
      # Add namespace as label
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
```

**Kubernetes SD Roles:**

| Role | Description | Use Case |
|------|-------------|----------|
| `node` | Kubernetes nodes | Node-level metrics |
| `pod` | Kubernetes pods | Application metrics |
| `service` | Kubernetes services | Service-level metrics |
| `endpoints` | Service endpoints | Pod-level via service |
| `ingress` | Kubernetes ingress | Ingress metrics |

### Consul Service Discovery

```yaml
scrape_configs:
  - job_name: 'consul-services'
    consul_sd_configs:
      - server: 'consul:8500'
        services: ['web', 'api', 'database']
        tags: ['production']
        node_meta:
          rack: 'rack-1'
    
    relabel_configs:
      - source_labels: [__meta_consul_service]
        target_label: job
      - source_labels: [__meta_consul_node]
        target_label: instance
```

### DNS Service Discovery

```yaml
scrape_configs:
  - job_name: 'dns-sd'
    dns_sd_configs:
      - names:
          - 'tasks.backend.example.com'
          - 'tasks.frontend.example.com'
        type: 'A'
        port: 9100
        refresh_interval: 30s
```

### EC2 Service Discovery

```yaml
scrape_configs:
  - job_name: 'ec2-instances'
    ec2_sd_configs:
      - region: us-east-1
        access_key: YOUR_ACCESS_KEY
        secret_key: YOUR_SECRET_KEY
        port: 9100
        filters:
          - name: tag:Environment
            values: ['production']
          - name: instance-state-name
            values: ['running']
    
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Name]
        target_label: instance
      - source_labels: [__meta_ec2_availability_zone]
        target_label: az
```

---

## Advanced Alerting with Prometheus

### Alert Rule Structure

```yaml
# alerting-rules.yml
groups:
  - name: application_alerts
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum by (job) (rate(http_requests_total[5m]))
          > 0.05
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "High error rate detected"
          description: "Job {{ $labels.job }} has error rate > 5% (current: {{ $value | humanizePercentage }})"
          runbook_url: "https://wiki.example.com/runbooks/high-error-rate"
      
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "P95 latency for {{ $labels.job }} is {{ $value | humanizeDuration }}"
```

### Alert Severity Levels

```yaml
labels:
  severity: critical    # Page immediately, 24/7
  severity: error       # Page during business hours
  severity: warning     # Ticket, investigate soon
  severity: info        # Log only, no action needed
```

### Alert Expressions Best Practices

```promql
# GOOD: Use rates for counters
rate(http_requests_total{status="500"}[5m])

# BAD: Don't use raw counters
http_requests_total{status="500"}

# GOOD: Use appropriate time windows
rate(http_requests_total[5m])

# BAD: Too short or too long
rate(http_requests_total[30s])   # Too noisy
rate(http_requests_total[1h])    # Too slow

# GOOD: Use thresholds with context
node_memory_MemAvailable_bytes < 100000000

# BETTER: Percentage-based
(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) < 0.1
```

### Alert Routing to Alertmanager

```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  slack_api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'

route:
  receiver: 'default-receiver'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true
    
    - match:
        severity: warning
      receiver: 'slack-warnings'
    
    - match:
        team: database
      receiver: 'database-team'

receivers:
  - name: 'default-receiver'
    email_configs:
      - to: 'oncall@example.com'
  
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_KEY'
  
  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warning'
        send_resolved: true
  
  - name: 'database-team'
    email_configs:
      - to: 'dba-team@example.com'
```

---

## Hands-on Lab: Complete Prometheus Setup

### Lab Objectives

By the end of this lab, you will:
1. Deploy Prometheus using Docker Compose
2. Configure multiple scrape targets
3. Write and execute PromQL queries
4. Create recording rules
5. Set up alerting rules
6. Visualize metrics in Prometheus UI

### Prerequisites

- Docker and Docker Compose installed
- Basic understanding of Linux commands
- Text editor (vim, nano, or VS Code)

### Step 1: Create Project Structure

```bash
mkdir -p prometheus-lab/{prometheus,grafana,exporters}
cd prometheus-lab
```

### Step 2: Create Prometheus Configuration

Create `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'prometheus-lab'
    environment: 'development'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

rule_files:
  - "rules/*.yml"

scrape_configs:
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  
  # Node Exporter for system metrics
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
  
  # Application metrics
  - job_name: 'app-metrics'
    static_configs:
      - targets: ['app:8080']
    metrics_path: /metrics
    
  # Blackbox exporter for endpoint probing
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://google.com
          - https://github.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

### Step 3: Create Recording Rules

Create `prometheus/rules/recording-rules.yml`:

```yaml
groups:
  - name: recording_rules
    interval: 30s
    rules:
      # Request rate by job
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
      
      # Error rate ratio
      - record: job:http_errors:ratio_rate5m
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum by (job) (rate(http_requests_total[5m]))
      
      # CPU utilization
      - record: instance:node_cpu_utilization:ratio_rate5m
        expr: |
          1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
      
      # Memory utilization
      - record: instance:node_memory_utilization:ratio
        expr: |
          1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
      
      # Disk utilization
      - record: instance:node_disk_utilization:ratio
        expr: |
          1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)
```

### Step 4: Create Alerting Rules

Create `prometheus/rules/alerting-rules.yml`:

```yaml
groups:
  - name: application_alerts
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          job:http_errors:ratio_rate5m > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.job }}"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"
      
      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High latency on {{ $labels.job }}"
          description: "P95 latency is {{ $value | humanizeDuration }}"
  
  - name: infrastructure_alerts
    rules:
      # High CPU usage
      - alert: HighCPUUsage
        expr: instance:node_cpu_utilization:ratio_rate5m > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | humanizePercentage }}"
      
      # Low memory available
      - alert: LowMemoryAvailable
        expr: instance:node_memory_utilization:ratio > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low memory on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanizePercentage }}"
      
      # Instance down
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute."
      
      # Disk space low
      - alert: DiskSpaceLow
        expr: instance:node_disk_utilization:ratio > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: "Disk usage is {{ $value | humanizePercentage }}"
```

### Step 5: Create Docker Compose File

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.45.0
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    networks:
      - monitoring
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:v1.6.0
    container_name: node-exporter
    command:
      - '--path.rootfs=/host'
    volumes:
      - '/:/host:ro,rslave'
    ports:
      - "9100:9100"
    networks:
      - monitoring
    restart: unless-stopped

  blackbox-exporter:
    image: prom/blackbox-exporter:v0.24.0
    container_name: blackbox-exporter
    volumes:
      - ./prometheus/blackbox.yml:/etc/blackbox/blackbox.yml:ro
    command:
      - '--config.file=/etc/blackbox/blackbox.yml'
    ports:
      - "9115:9115"
    networks:
      - monitoring
    restart: unless-stopped

  app:
    image: nginx:alpine
    container_name: sample-app
    volumes:
      - ./app/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./app/metrics.lua:/usr/share/lua/5.1/metrics.lua:ro
    ports:
      - "8080:80"
    networks:
      - monitoring
    restart: unless-stopped

volumes:
  prometheus-data:

networks:
  monitoring:
    driver: bridge
```

### Step 6: Create Sample App with Metrics

Create `app/nginx.conf`:

```nginx
load_module modules/ngx_http_lua_module.so;

events {
    worker_connections 1024;
}

http {
    lua_package_path "/usr/share/lua/5.1/?.lua;;";
    
    init_worker_by_lua_block {
        metrics = require("metrics")
        metrics.init()
    }
    
    server {
        listen 80;
        
        location /metrics {
            content_by_lua_block {
                metrics.collect()
                ngx.say(metrics.prometheus())
            }
        }
        
        location / {
            content_by_lua_block {
                metrics.inc_request()
                local start = ngx.now()
                
                -- Simulate some work
                ngx.sleep(math.random(10, 100) / 1000)
                
                local duration = ngx.now() - start
                metrics.observe_request_duration(duration)
                
                ngx.say("Hello from monitored app!")
            }
        }
        
        location /health {
            return 200 "OK\n";
        }
    }
}
```

Create `app/metrics.lua`:

```lua
local _M = {}

local requests_total = 0
local request_durations = {}

function _M.init()
    requests_total = 0
    request_durations = {}
end

function _M.inc_request()
    requests_total = requests_total + 1
end

function _M.observe_request_duration(duration)
    table.insert(request_durations, duration)
end

function _M.collect()
    -- Collect metrics logic
end

function _M.prometheus()
    local output = ""
    output = output .. "# HELP http_requests_total Total HTTP requests\n"
    output = output .. "# TYPE http_requests_total counter\n"
    output = output .. "http_requests_total " .. requests_total .. "\n"
    
    output = output .. "# HELP http_request_duration_seconds HTTP request duration\n"
    output = output .. "# TYPE http_request_duration_seconds histogram\n"
    
    -- Simplified histogram
    local buckets = {0.01, 0.05, 0.1, 0.5, 1.0}
    local counts = {}
    for _, bucket in ipairs(buckets) do
        counts[bucket] = 0
    end
    
    for _, duration in ipairs(request_durations) do
        for _, bucket in ipairs(buckets) do
            if duration <= bucket then
                counts[bucket] = counts[bucket] + 1
            end
        end
    end
    
    local cumulative = 0
    for _, bucket in ipairs(buckets) do
        cumulative = cumulative + counts[bucket]
        output = output .. string.format(
            "http_request_duration_seconds_bucket{le=\"%s\"} %d\n",
            bucket, cumulative
        )
    end
    
    output = output .. string.format(
        "http_request_duration_seconds_count %d\n",
        #request_durations
    )
    
    local sum = 0
    for _, d in ipairs(request_durations) do
        sum = sum + d
    end
    output = output .. string.format(
        "http_request_duration_seconds_sum %.3f\n",
        sum
    )
    
    return output
end

return _M
```

### Step 7: Start the Stack

```bash
docker-compose up -d
```

### Step 8: Verify Targets

Open browser to `http://localhost:9090/targets`

All targets should show as "UP":
- prometheus
- node-exporter
- app-metrics
- blackbox

### Step 9: Execute PromQL Queries

Navigate to `http://localhost:9090/graph` and try these queries:

#### Basic Queries
```promql
# All time series
up

# Specific job
node_cpu_seconds_total{mode="idle"}

# Rate of requests
rate(http_requests_total[5m])
```

#### Aggregations
```promql
# Total requests by job
sum by (job) (rate(http_requests_total[5m]))

# Average CPU usage
avg(1 - rate(node_cpu_seconds_total{mode="idle"}[5m]))

# Top 3 instances by memory
topk(3, node_memory_MemTotal_bytes)
```

#### Advanced Queries
```promql
# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m])) 
/ 
sum(rate(http_requests_total[5m])) 
* 100

# P95 latency
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# Predict disk full
predict_linear(node_filesystem_avail_bytes[1h], 24*3600)
```

### Step 10: View Rules

Check recording rules: `http://localhost:9090/rules`

Check alerting rules: `http://localhost:9090/alerts`

### Step 11: Cleanup

```bash
docker-compose down -v
```

---

## Best Practices & Common Pitfalls

### ✅ Best Practices

#### Configuration
1. **Use configuration management** (Ansible, Terraform) for Prometheus config
2. **Version control** all configuration files
3. **Test configurations** before deploying (`promtool check config`)
4. **Use separate rule files** for organization
5. **Document all recording and alerting rules**

#### Performance
1. **Appropriate scrape intervals** based on metric volatility
2. **Limit time series cardinality** (< 100k per Prometheus instance)
3. **Use recording rules** for expensive queries
4. **Implement federation** for large-scale deployments
5. **Monitor Prometheus itself** (eat your own dog food)

#### Storage
1. **Size storage appropriately** (retention × ingestion rate)
2. **Use SSDs** for better performance
3. **Implement backup strategy** for critical data
4. **Monitor disk usage** with alerts
5. **Plan for growth** (20-30% buffer)

#### Querying
1. **Use specific label matchers** to reduce scan scope
2. **Avoid regex** when exact match works
3. **Choose appropriate time windows** for rate calculations
4. **Test queries** before using in dashboards/alerts
5. **Document complex queries**

### ❌ Common Pitfalls

#### Cardinality Explosion
```promql
# BAD: High cardinality labels
http_requests_total{user_id="12345", url="/users/12345/profile"}

# GOOD: Aggregate appropriately
http_requests_total{endpoint="/users/profile", method="GET"}
```

#### Incorrect Rate Calculations
```promql
# BAD: Using raw counters
http_requests_total

# GOOD: Use rate for counters
rate(http_requests_total[5m])
```

#### Too Short/Long Time Windows
```promql
# BAD: Too noisy
rate(http_requests_total[30s])

# BAD: Too slow to react
rate(http_requests_total[1h])

# GOOD: Balanced
rate(http_requests_total[5m])
```

#### Missing Label Matchers
```promql
# BAD: Scans all series
sum(rate(http_requests_total[5m]))

# GOOD: Filter by job
sum by (job) (rate(http_requests_total{job="api"}[5m]))
```

#### Alert Fatigue
```yaml
# BAD: Too sensitive
- alert: HighCPU
  expr: node_cpu_utilization > 0.5
  for: 1m

# GOOD: Reasonable threshold and duration
- alert: HighCPU
  expr: node_cpu_utilization > 0.8
  for: 5m
```

---

## Troubleshooting Guide

### Issue: Targets Showing as DOWN

**Symptoms:**
- Targets page shows red "DOWN" status
- No metrics being collected

**Diagnosis:**
```bash
# Check Prometheus logs
docker logs prometheus

# Test connectivity
curl http://target-host:9100/metrics

# Check network
ping target-host
```

**Solutions:**
1. Verify target is running and accessible
2. Check firewall rules
3. Verify correct port in scrape config
4. Check authentication requirements
5. Review relabeling rules

### Issue: High Memory Usage

**Symptoms:**
- Prometheus consuming excessive RAM
- OOM kills occurring

**Diagnosis:**
```promql
# Check series count
count({__name__=~".+"})

# Check ingestion rate
sum(rate(prometheus_tsdb_head_samples_appended_total[5m]))
```

**Solutions:**
1. Reduce scrape interval for non-critical metrics
2. Drop high-cardinality metrics with relabeling
3. Implement metric filtering
4. Scale horizontally with federation
5. Increase retention time cautiously

### Issue: Slow Queries

**Symptoms:**
- Dashboard loading slowly
- Query timeout errors

**Diagnosis:**
```promql
# Check query duration
prometheus_http_request_duration_seconds

# Identify expensive queries
topk(10, sort_desc(sum by (handler) (rate(prometheus_http_request_duration_seconds_sum[5m]))))
```

**Solutions:**
1. Create recording rules for complex queries
2. Add appropriate label matchers
3. Reduce time range for queries
4. Optimize regex patterns
5. Increase query timeout if necessary

### Issue: Missing Metrics

**Symptoms:**
- Expected metrics not appearing
- Gaps in time series

**Diagnosis:**
```bash
# Check exporter logs
docker logs node-exporter

# Verify metric exposition
curl http://exporter:9100/metrics | grep metric_name
```

**Solutions:**
1. Verify exporter version compatibility
2. Check metric name changes between versions
3. Review scrape configuration
4. Check for relabeling dropping metrics
5. Verify metric help text for correct name

### Issue: Alert Not Firing

**Symptoms:**
- Condition met but no alert
- Alert rules showing as inactive

**Diagnosis:**
```promql
# Test alert expression manually
<your-alert-expression>

# Check rule evaluation
prometheus_rule_evaluation_duration_seconds
prometheus_rule_evaluation_failures_total
```

**Solutions:**
1. Verify expression syntax
2. Check `for` duration requirement
3. Review label matchers in routing
4. Check Alertmanager connectivity
5. Verify inhibition rules not suppressing

---

## Summary

This deep dive covered:

✅ **Architecture**: Core components and data flow
✅ **Data Model**: Metric types, labels, and best practices
✅ **PromQL**: From basics to advanced query patterns
✅ **Recording Rules**: Pre-computation for performance
✅ **Service Discovery**: Multiple mechanisms for dynamic environments
✅ **Alerting**: Comprehensive alerting strategies
✅ **Hands-on Lab**: Complete working Prometheus setup
✅ **Best Practices**: Production-ready configurations
✅ **Troubleshooting**: Common issues and solutions

### Next Steps

1. Practice writing PromQL queries daily
2. Implement recording rules for your most-used queries
3. Set up comprehensive alerting with proper routing
4. Monitor Prometheus itself
5. Explore advanced topics: Federation, Thanos, Cortex

### Resources

- **Official Docs**: https://prometheus.io/docs/
- **PromQL Cheat Sheet**: https://promlabs.com/promql-cheat-sheet/
- **Awesome Prometheus**: https://awesome-prometheus-alerts.grep.to/
- **Community**: https://prometheus.io/community/

---

**Ready for Section 03: Grafana Dashboards & Visualization** 📊
