# Section 01: Introduction to Monitoring & Observability

## 📖 Table of Contents
1. [Monitoring vs Observability](#monitoring-vs-observability)
2. [The Three Pillars of Observability](#the-three-pillars-of-observability)
3. [Golden Signals of Monitoring](#golden-signals-of-monitoring)
4. [Monitoring vs Alerting](#monitoring-vs-alerting)
5. [Modern Observability Stack](#modern-observability-stack)
6. [Hands-on Exercise](#hands-on-exercise)

---

## Monitoring vs Observability

### What is Monitoring?

**Monitoring** is the process of collecting, analyzing, and using data to track the health and performance of systems. It focuses on:

- **Known unknowns**: Things you know can go wrong
- **Pre-defined metrics**: CPU usage, memory, disk space, request counts
- **Threshold-based alerts**: "Alert when CPU > 80%"
- **Reactive approach**: Responding to known failure modes

**Traditional Monitoring Questions:**
- Is the system up or down?
- Is the CPU usage normal?
- Are requests being served?
- Is the error rate within acceptable limits?

### What is Observability?

**Observability** is a measure of how well internal states of a system can be inferred from knowledge of its external outputs. It enables you to:

- **Unknown unknowns**: Investigate issues you didn't anticipate
- **Ad-hoc exploration**: Ask new questions without deploying new code
- **Context-rich debugging**: Understand the "why" behind failures
- **Proactive approach**: Discover issues before they impact users

**Observability Questions:**
- Why is this request slow?
- What changed before the error started?
- How are services interacting?
- What's the user experience for this specific segment?

### Key Differences

| Aspect | Monitoring | Observability |
|--------|-----------|---------------|
| **Focus** | Known failure modes | Unknown issues |
| **Approach** | Pre-defined metrics | Exploratory analysis |
| **Questions** | "Is it working?" | "Why is it broken?" |
| **Tooling** | Dashboards, alerts | Tracing, log exploration |
| **Mindset** | Reactive | Proactive & investigative |
| **Data** | Aggregated metrics | High-cardinality events |

### The Evolution

```
Traditional Monitoring (2000s)
    ↓
Application Performance Management (2010s)
    ↓
Cloud-Native Monitoring (2015s)
    ↓
Observability (2018+)
```

**Why the shift?**
- Microservices architecture increased complexity
- Dynamic infrastructure (containers, Kubernetes)
- Faster deployment cycles
- Need for faster mean-time-to-resolution (MTTR)

---

## The Three Pillars of Observability

### 1. Metrics

**Metrics** are numerical measurements collected over time that represent the state of a system.

#### Characteristics:
- **Aggregated**: Summarized data (averages, percentiles, counts)
- **Time-series**: Values associated with timestamps
- **Efficient**: Low storage overhead, fast queries
- **Structured**: Pre-defined schema

#### Metric Types:

**Counters**
- Monotonically increasing values
- Examples: Total requests, errors, bytes sent
```promql
# Prometheus counter example
http_requests_total{method="POST", status="200"}
```

**Gauges**
- Values that can go up or down
- Examples: CPU usage, memory, temperature
```promql
# Prometheus gauge example
node_memory_MemAvailable_bytes
```

**Histograms**
- Distribution of values across buckets
- Examples: Request latency, response sizes
```promql
# Prometheus histogram example
http_request_duration_seconds_bucket{le="0.5"}
```

**Summaries**
- Similar to histograms but calculate quantiles
- Examples: p95, p99 latencies
```promql
# Summary example
http_request_duration_seconds{quantile="0.99"}
```

#### Best Practices for Metrics:
- Use consistent naming conventions
- Include relevant labels/dimensions
- Avoid high-cardinality labels (user IDs, IPs)
- Define clear units (seconds, bytes, requests)
- Document metric meanings and thresholds

### 2. Logs

**Logs** are immutable, timestamped records of discrete events that occurred in a system.

#### Log Levels:

| Level | Description | Use Case |
|-------|-------------|----------|
| **DEBUG** | Detailed diagnostic information | Development, troubleshooting |
| **INFO** | General operational messages | Normal operations |
| **WARN** | Potential issues | Degraded but functional |
| **ERROR** | Actual errors | Failed operations |
| **FATAL** | Critical failures | System shutdown |

#### Log Structure:

**Unstructured Logs:**
```
2024-01-15 10:30:45 ERROR Database connection failed for user john
```

**Structured Logs (JSON):**
```json
{
  "timestamp": "2024-01-15T10:30:45Z",
  "level": "ERROR",
  "service": "user-service",
  "message": "Database connection failed",
  "user_id": "john",
  "error_code": "DB_CONN_001",
  "trace_id": "abc123xyz",
  "host": "pod-user-service-7d8f9c"
}
```

#### Log Collection Strategies:

**Agent-based:**
- Install log shipper on each node (Filebeat, Fluent Bit)
- Pros: Centralized management, efficient
- Cons: Additional resource usage

**Sidecar Pattern:**
- Deploy log collector alongside each application container
- Pros: Isolation, per-pod configuration
- Cons: More resources, complexity

**Direct Export:**
- Applications send logs directly to backend
- Pros: No additional components
- Cons: Application coupling, retry logic needed

#### Best Practices for Logging:
- Use structured logging (JSON)
- Include correlation IDs (trace_id, span_id)
- Avoid logging sensitive data (PII, passwords)
- Implement log rotation and retention policies
- Standardize log formats across services
- Log context, not just errors

### 3. Traces

**Traces** track the flow of a request through a distributed system, showing how services interact.

#### Trace Hierarchy:

```
Trace (entire request journey)
├── Span 1: API Gateway (0-50ms)
│   ├── Span 2: Auth Service (5-25ms)
│   └── Span 3: User Service (30-45ms)
│       └── Span 4: Database Query (35-42ms)
└── Span 5: Cache Lookup (10-15ms)
```

#### Key Concepts:

**Trace ID**: Unique identifier for the entire request
**Span ID**: Unique identifier for a single operation
**Parent Span ID**: Links child spans to parents
**Tags/Attributes**: Key-value metadata (HTTP method, status code)
**Events**: Timestamped annotations within a span

#### Distributed Context Propagation:

**HTTP Headers:**
```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
tracestate: vendor1=value1,vendor2=value2
```

**Propagation Formats:**
- **W3C Trace Context**: Standard format
- **B3**: Zipkin/B3 format (used by Zipkin, Jaeger)
- **AWS X-Ray**: Amazon's format

#### Instrumentation Approaches:

**Automatic Instrumentation:**
- Agent-based (Java agent, Python wrapper)
- Library-level (middleware, framework plugins)
- Pros: Minimal code changes
- Cons: May miss custom logic

**Manual Instrumentation:**
- Explicit trace creation in code
- Custom span creation
- Pros: Full control, business logic visibility
- Cons: Code changes required

**Example (OpenTelemetry Python):**
```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

@tracer.start_as_current_span("process_order")
def process_order(order_id):
    with tracer.start_as_current_span("validate_payment"):
        validate_payment(order_id)
    
    with tracer.start_as_current_span("update_inventory"):
        update_inventory(order_id)
```

#### Best Practices for Tracing:
- Instrument all services consistently
- Propagate context across service boundaries
- Include business-relevant attributes
- Sample strategically (100% for errors, lower for success)
- Correlate traces with logs and metrics
- Monitor tracing overhead (<5% performance impact)

---

## Golden Signals of Monitoring

The **Four Golden Signals** were defined by Google's Site Reliability Engineering team as the most important metrics to monitor:

### 1. Latency

**Definition**: Time taken to service a request

**Key Measurements:**
- Average latency (can be misleading)
- Percentiles (p50, p90, p95, p99)
- Tail latency (p99, p99.9)

**Why Percentiles Matter:**
```
Request Latencies (ms): [10, 12, 11, 13, 15, 14, 1000, 12, 11, 13]

Average: 111.1 ms  ← Misleading!
p50: 12.5 ms       ← Typical user experience
p95: 1000 ms       ← Worst 5% of users
p99: 1000 ms       ← Worst 1% of users
```

**Best Practices:**
- Monitor multiple percentiles (p50, p95, p99)
- Set SLOs based on p95 or p99
- Break down by endpoint, method, user segment
- Track both server-side and client-side latency

### 2. Traffic

**Definition**: Demand placed on your system

**Common Metrics:**
- Requests per second (RPS)
- Concurrent connections
- Network bandwidth
- Queue depth

**Examples:**
```promql
# HTTP requests per second
rate(http_requests_total[5m])

# Active connections
nginx_connections_active

# Kafka consumer lag
kafka_consumer_group_lag
```

**Best Practices:**
- Monitor traffic patterns (daily/weekly seasonality)
- Set up anomaly detection for unusual spikes/drops
- Track traffic by source, endpoint, region
- Correlate traffic with deployments and marketing events

### 3. Errors

**Definition**: Rate of requests that fail

**Error Types:**
- **Explicit errors**: HTTP 5xx, exception throws
- **Implicit errors**: Slow responses, incorrect results
- **Partial errors**: Degraded functionality

**Measurement Approaches:**
```promql
# Error rate (HTTP 5xx)
rate(http_requests_total{status=~"5.."}[5m]) 
/ 
rate(http_requests_total[5m])

# Error budget consumption
1 - (sum(rate(successful_requests[30d])) / sum(rate(total_requests[30d])))
```

**Best Practices:**
- Track error rates, not just error counts
- Distinguish between client (4xx) and server (5xx) errors
- Monitor error budgets for SLO compliance
- Alert on error rate increases, not absolute thresholds

### 4. Saturation

**Definition**: How "full" your resources are

**Resources to Monitor:**
- CPU utilization
- Memory usage
- Disk space and I/O
- Network bandwidth
- Connection pools
- Thread pools

**Leading vs Lagging Indicators:**

**Lagging** (already saturated):
- CPU at 100%
- Out of memory errors
- Disk full alerts

**Leading** (predicting saturation):
- Memory growth trend
- Queue length increase
- Connection pool exhaustion rate

**Best Practices:**
- Monitor utilization AND capacity trends
- Set alerts before resources are exhausted
- Track saturation by service tier (critical vs non-critical)
- Plan capacity based on growth projections

### The Fourth Signal (Bonus): Churn

Some teams add a fifth signal:

**Churn**: Rate of instance restarts, deployments, or configuration changes

```promql
# Pod restarts
increase(kube_pod_container_status_restarts_total[1h])

# Deployment frequency
count(deployments_created_total[1d])
```

---

## Monitoring vs Alerting

### Understanding the Difference

**Monitoring** = Collecting and visualizing data
**Alerting** = Notifying humans when action is needed

### The Alerting Pyramid

```
         /\
        /  \       Pages (Immediate action required)
       /----\      24/7 on-call, wake someone up
      /      \     
     /--------\    Tickets (Action needed soon)
    /          \   Business hours response
   /------------\  
  /              \  Logs (Investigation only)
 /----------------\ No immediate action
```

### Alert Types

**Page-worthy Alerts:**
- Customer-facing impact
- Data loss risk
- Security breach
- Complete service outage
- SLO burn rate critical

**Ticket-worthy Alerts:**
- Degraded performance
- Partial failures
- Capacity warnings
- Non-critical errors

**Log-only Events:**
- Recovered automatically
- Expected transient failures
- Informational events
- Debug data

### Writing Effective Alert Rules

**Bad Alert:**
```yaml
alert: HighCPU
expr: cpu_usage > 80
for: 1m
annotations:
  summary: "CPU is high"
```

**Good Alert:**
```yaml
alert: HighCPUSustained
expr: avg_over_time(cpu_usage[5m]) > 80
for: 10m
labels:
  severity: warning
  team: platform
annotations:
  summary: "Sustained high CPU on {{ $labels.instance }}"
  description: "CPU usage has been above 80% for 10 minutes. Current value: {{ $value }}%"
  runbook_url: "https://wiki.company.com/runbooks/high-cpu"
```

### Alert Fatigue Prevention

**Symptoms:**
- Ignoring alerts
- Delayed response times
- Alert muting
- On-call burnout

**Solutions:**
1. **Reduce noise**: Only alert on actionable items
2. **Aggregate alerts**: Group related symptoms
3. **Use inhibition**: Suppress dependent alerts
4. **Implement escalation**: Tiered response
5. **Regular review**: Weekly alert hygiene
6. **Measure effectiveness**: Track false positive rate

### Alert Routing Strategy

```yaml
# Alertmanager configuration example
route:
  receiver: default
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  
  routes:
    - match:
        severity: critical
      receiver: pagerduty-critical
      continue: true
      
    - match:
        team: database
      receiver: db-team-slack
      
    - match:
        environment: production
      receiver: prod-oncall
```

---

## Modern Observability Stack

### Open Source Stack

```
┌─────────────────────────────────────────┐
│           Visualization Layer           │
│         Grafana | Kibana | Jaeger       │
└─────────────────────────────────────────┘
                    ↑
┌─────────────────────────────────────────┐
│            Query & Analysis             │
│    PromQL | ES Query | Trace Query      │
└─────────────────────────────────────────┘
                    ↑
┌─────────────────────────────────────────┐
│            Storage Layer                │
│  Prometheus | Elasticsearch | Jaeger    │
└─────────────────────────────────────────┘
                    ↑
┌─────────────────────────────────────────┐
│          Collection & Processing        │
│  Node Exporter | Fluent Bit | OTEL     │
└─────────────────────────────────────────┘
                    ↑
┌─────────────────────────────────────────┐
│         Applications & Infrastructure    │
│    Microservices | Containers | Cloud   │
└─────────────────────────────────────────┘
```

### Commercial Solutions

| Category | Tools |
|----------|-------|
| **All-in-One** | Datadog, New Relic, Dynatrace, Splunk |
| **Metrics** | Datadog, New Relic, Splunk Infrastructure |
| **Logging** | Splunk, Datadog Logs, Elastic Cloud |
| **Tracing** | Datadog APM, New Relic APM, Lightstep |
| **RUM** | New Relic Browser, Datadog RUM, SpeedCurve |

### Cloud-Native Options

**AWS:**
- CloudWatch (metrics/logs)
- X-Ray (tracing)
- Managed Prometheus & Grafana

**Google Cloud:**
- Cloud Monitoring (formerly Stackdriver)
- Cloud Trace
- Cloud Logging

**Azure:**
- Azure Monitor
- Application Insights
- Log Analytics

### Choosing Your Stack

**Consider:**
1. **Scale**: Data volume, cardinality requirements
2. **Budget**: Open source vs commercial TCO
3. **Team skills**: Existing expertise
4. **Integration**: Compatibility with current tools
5. **Compliance**: Data residency, retention requirements
6. **Support**: Community vs enterprise support

**Recommended Starting Stack:**
```yaml
Metrics: Prometheus + Grafana
Logging: EFK (Elasticsearch, Fluent Bit, Kibana)
Tracing: Jaeger or Tempo
APM: OpenTelemetry + Backend of choice
Alerting: Alertmanager + PagerDuty/OpsGenie
```

---

## Hands-on Exercise

### Exercise 1: Install Prometheus and Grafana

**Prerequisites:**
- Docker installed
- Basic Linux command line knowledge

**Step 1: Create docker-compose.yml**
```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.enable-lifecycle'

  grafana:
    image: grafana/grafana:10.1.0
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:v1.6.1
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'

volumes:
  prometheus_data:
  grafana_data:
```

**Step 2: Create prometheus.yml**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

**Step 3: Start the stack**
```bash
docker-compose up -d
```

**Step 4: Access the interfaces**
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000 (admin/admin)
- Node Exporter Metrics: http://localhost:9100/metrics

**Step 5: Configure Grafana**
1. Login to Grafana
2. Go to Configuration → Data Sources
3. Add data source → Prometheus
4. URL: http://prometheus:9090
5. Save & Test

**Step 6: Create your first dashboard**
1. Go to Dashboards → Create → New Dashboard
2. Add visualization
3. Try these queries:
```promql
# CPU Usage
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory Usage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Disk Usage
(node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes * 100
```

### Exercise 2: Explore the Three Pillars

**Metrics Exploration:**
1. Open Prometheus UI
2. Try these queries:
```promql
# Count of scrapes
count(up)

# Scrape duration
scrape_duration_seconds

# Up/down status
up
```

**Log Exploration:**
1. Generate sample logs:
```bash
docker logs node-exporter
```
2. Notice the structured output
3. Practice filtering by level

**Trace Simulation:**
Since we don't have tracing yet, simulate with curl timing:
```bash
# Measure request latency
time curl http://localhost:9090/api/v1/query?query=up

# Check response headers
curl -I http://localhost:9090
```

### Exercise 3: Calculate Golden Signals

Create a file `golden_signals.yml`:
```yaml
groups:
  - name: golden_signals
    rules:
      # Latency (p95)
      - record: http_request_latency_p95
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
      
      # Traffic (RPS)
      - record: http_requests_per_second
        expr: rate(http_requests_total[5m])
      
      # Error Rate
      - record: http_error_rate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])
      
      # Saturation (Memory)
      - record: memory_saturation
        expr: 1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```

Load into Prometheus:
```bash
# Update prometheus.yml to include rule files
rule_files:
  - "golden_signals.yml"

# Reload Prometheus
curl -X POST http://localhost:9090/-/reload
```

### Reflection Questions

1. What's the difference between monitoring your home WiFi vs observability of a microservices application?
2. Why are percentiles more useful than averages for latency?
3. When should you page someone vs create a ticket?
4. How would you explain the three pillars to a non-technical stakeholder?

---

## Key Takeaways

✅ **Monitoring** tracks known issues; **Observability** helps investigate unknown issues

✅ **Three Pillars**: Metrics (numbers), Logs (events), Traces (request flows)

✅ **Golden Signals**: Latency, Traffic, Errors, Saturation

✅ **Alert Wisely**: Only alert on actionable items, avoid alert fatigue

✅ **Choose Tools Based On**: Scale, budget, team skills, integration needs

---

## Next Steps

➡️ **Section 02**: Deep dive into Prometheus for metrics collection
➡️ **Lab 01**: Deploy complete Prometheus + Grafana stack
➡️ **Practice**: Calculate golden signals for your own applications

---

## Additional Resources

- [Google SRE Book - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Grafana Labs Learning Portal](https://grafana.com/learn/)
- [Awesome Observability GitHub Repo](https://github.com/open-telemetry/awesome-opentelemetry)
