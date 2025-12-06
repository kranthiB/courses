# KCNA Exam Study Guide: Domain 4 - Cloud Native Observability

## Domain Weight: 8% (~5 questions out of 60)

This domain covers the principles of observability, including monitoring, logging, tracing, and the tools used in cloud native environments.

---

## 4.1 What is Observability?

### Definition
Observability is the ability to understand the internal state of a system by examining its external outputs. It answers the question: "What is happening inside my system right now?"

### Monitoring vs Observability

| Aspect | Monitoring | Observability |
|--------|------------|---------------|
| **Approach** | Predefined metrics and alerts | Exploratory, ask new questions |
| **Questions** | Known unknowns | Unknown unknowns |
| **Scope** | Specific metrics | System-wide understanding |
| **Use Case** | Track expected failures | Debug unexpected issues |

### Why Observability Matters in Cloud Native
- **Distributed systems**: Microservices increase complexity
- **Dynamic environments**: Containers are ephemeral
- **Scale**: Thousands of containers and services
- **Debugging**: Need to trace requests across services
- **Performance**: Identify bottlenecks quickly

---

## 4.2 Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY                            │
│                                                             │
│    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│    │             │  │             │  │             │        │
│    │   METRICS   │  │    LOGS     │  │   TRACES    │        │
│    │             │  │             │  │             │        │
│    │  "What is   │  │  "What      │  │  "How did   │        │
│    │  happening?"│  │  happened?" │  │  it happen?"│        │
│    │             │  │             │  │             │        │
│    └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.2.1 Metrics

**Definition**: Numerical measurements collected at regular intervals

**Characteristics**:
- Quantitative data (numbers)
- Time-series data (values over time)
- Aggregatable (can be combined, averaged)
- Low cardinality (limited unique values)

**Types of Metrics**:
| Type | Description | Example |
|------|-------------|---------|
| **Counter** | Only increases | Total requests, errors |
| **Gauge** | Can increase or decrease | CPU usage, memory usage |
| **Histogram** | Distribution of values | Request latency distribution |
| **Summary** | Similar to histogram with quantiles | Request duration percentiles |

**Key Metrics to Monitor**:
- **Golden Signals** (Google SRE):
  - **Latency**: How long requests take
  - **Traffic**: How much demand is on the system
  - **Errors**: Rate of failing requests
  - **Saturation**: How "full" the system is

- **RED Method** (Microservices):
  - **Rate**: Requests per second
  - **Errors**: Failed requests
  - **Duration**: Request latency

- **USE Method** (Resources):
  - **Utilization**: Percentage of time resource is busy
  - **Saturation**: Degree of queued work
  - **Errors**: Error events

### 4.2.2 Logs

**Definition**: Immutable, timestamped records of discrete events

**Characteristics**:
- Unstructured or structured text
- Rich in context
- High volume
- Event-driven

**Log Levels**:
| Level | Description | Use Case |
|-------|-------------|----------|
| **DEBUG** | Detailed debugging information | Development |
| **INFO** | General operational events | Normal operation |
| **WARN** | Something unexpected but not critical | Potential issues |
| **ERROR** | Error events, service still running | Failures |
| **FATAL** | Critical errors, service may stop | Critical failures |

**Structured Logging**:
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "ERROR",
  "service": "payment-service",
  "trace_id": "abc123",
  "message": "Payment failed",
  "error": "insufficient_funds",
  "user_id": "user_456"
}
```

### 4.2.3 Traces (Distributed Tracing)

**Definition**: Records the path of a request as it flows through services

**Concepts**:
- **Trace**: Complete journey of a request
- **Span**: Individual operation within a trace
- **Trace ID**: Unique identifier for entire trace
- **Span ID**: Unique identifier for each operation
- **Parent Span**: The calling span

**Trace Visualization**:
```
Trace ID: abc-123

├── [Span 1] API Gateway (10ms)
│   └── [Span 2] User Service (5ms)
│       └── [Span 3] Database Query (2ms)
│   └── [Span 4] Order Service (15ms)
│       └── [Span 5] Payment Service (8ms)
│       └── [Span 6] Inventory Service (3ms)
```

**Use Cases for Tracing**:
- Debug slow requests
- Identify service dependencies
- Find performance bottlenecks
- Root cause analysis

---

## 4.3 Observability Tools and CNCF Projects

### Prometheus (CNCF Graduated)

**Purpose**: Metrics collection and alerting

**Key Features**:
- Pull-based model (scrapes metrics endpoints)
- Time-series database
- Powerful query language (PromQL)
- Built-in alerting (with Alertmanager)
- Service discovery

**Architecture**:
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Target 1   │    │  Target 2   │    │  Target 3   │
│ /metrics    │    │ /metrics    │    │ /metrics    │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       │    Scrape        │                  │
       └──────────────────┼──────────────────┘
                          │
                          ▼
                ┌─────────────────┐
                │   Prometheus    │
                │   Server        │
                │ (TSDB + Query)  │
                └────────┬────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   ┌───────────┐  ┌───────────┐  ┌───────────┐
   │ Alertmgr  │  │  Grafana  │  │  API/     │
   │           │  │           │  │  Tools    │
   └───────────┘  └───────────┘  └───────────┘
```

**PromQL Examples**:
```promql
# CPU usage percentage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# HTTP error rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# Memory usage
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

### Grafana

**Purpose**: Visualization and dashboarding

**Features**:
- Multi-source dashboards (Prometheus, Loki, etc.)
- Rich visualizations
- Alerting
- Annotations and annotations

**Common Use**:
- Prometheus for data → Grafana for visualization

### Fluentd / Fluent Bit (CNCF Graduated)

**Purpose**: Log collection and forwarding

**Fluentd**:
- Full-featured log collector
- Rich plugin ecosystem
- Ruby-based

**Fluent Bit**:
- Lightweight log processor
- Low memory footprint
- C-based
- Designed for edge/embedded systems

**Log Pipeline**:
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Container  │    │  Fluent Bit │    │ Destination │
│    Logs     │───►│  (Collect)  │───►│ (Elastic/   │
│             │    │             │    │  Loki/S3)   │
└─────────────┘    └─────────────┘    └─────────────┘
```

### Jaeger (CNCF Graduated)

**Purpose**: Distributed tracing

**Features**:
- OpenTracing compatible
- Service dependency analysis
- Root cause analysis
- Performance/latency optimization

**Components**:
- **Agent**: Receives spans from applications
- **Collector**: Receives spans from agents
- **Query**: UI and API for querying
- **Storage**: Backend (Cassandra, Elasticsearch)

### OpenTelemetry (CNCF Incubating → Graduated)

**Purpose**: Unified observability framework

**What it provides**:
- Standard APIs for metrics, logs, and traces
- SDKs for multiple languages
- Collector for receiving, processing, exporting data
- Vendor-neutral

**Components**:
- **API**: Instrumentation interfaces
- **SDK**: Implementation of the API
- **Collector**: Receives, processes, exports telemetry
- **Exporters**: Send data to backends

**Architecture**:
```
┌─────────────────────────────────────────────────────────────┐
│                    Application                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │         OpenTelemetry SDK                           │    │
│  │   (Traces + Metrics + Logs)                         │    │
│  └───────────────────────┬─────────────────────────────┘    │
└──────────────────────────┼──────────────────────────────────┘
                           │ OTLP
                           ▼
              ┌─────────────────────────┐
              │  OpenTelemetry          │
              │  Collector              │
              │  (Receivers, Processors,│
              │   Exporters)            │
              └───────────┬─────────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
   ┌──────────┐    ┌──────────┐    ┌──────────┐
   │ Jaeger   │    │Prometheus│    │  Vendor  │
   │ (Traces) │    │(Metrics) │    │  Backend │
   └──────────┘    └──────────┘    └──────────┘
```

### Grafana Loki

**Purpose**: Log aggregation system

**Key Features**:
- Like Prometheus, but for logs
- Only indexes metadata (labels), not full text
- Cost-effective storage
- LogQL query language
- Integrates seamlessly with Grafana

**Difference from Elasticsearch**:
| Aspect | Loki | Elasticsearch |
|--------|------|---------------|
| **Indexing** | Labels only | Full-text |
| **Storage** | Efficient | More expensive |
| **Query** | LogQL | Lucene/KQL |
| **Use Case** | Cloud native | Full-text search |

---

## 4.4 Kubernetes Native Observability

### Built-in Observability

**Metrics Server**:
- Lightweight metrics for HPA/VPA
- Provides CPU and memory metrics
- Not for long-term storage

**kubectl Commands for Observability**:
```bash
# View pod logs
kubectl logs <pod-name>
kubectl logs -f <pod-name>                # Stream logs
kubectl logs <pod-name> -c <container>    # Specific container
kubectl logs <pod-name> --previous        # Previous instance

# View resource usage
kubectl top nodes
kubectl top pods

# Describe resources (events, conditions)
kubectl describe pod <pod-name>
kubectl get events

# Debug with ephemeral containers
kubectl debug <pod-name> -it --image=busybox
```

### Kubernetes Events
- Record of what happened in the cluster
- Important for debugging
- Short-lived (default 1 hour retention)

```bash
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -w  # Watch for new events
```

### Container Probes for Health
- **Liveness Probe**: Is the container alive?
- **Readiness Probe**: Is the container ready to receive traffic?
- **Startup Probe**: Has the container started?

---

## 4.5 Telemetry Concepts

### What is Telemetry?
The automated collection and transmission of data from remote sources.

### Telemetry Data Types
1. **Metrics**: Aggregated numerical data
2. **Logs**: Event records
3. **Traces**: Request path through services
4. **Events**: Significant occurrences
5. **Profiling**: Code-level performance data

### Push vs Pull Models

**Pull Model (Prometheus)**:
- Server scrapes targets at intervals
- Targets expose metrics endpoint
- Good for short-lived jobs (need PushGateway)

**Push Model (OpenTelemetry)**:
- Applications push data to collector
- Better for ephemeral workloads
- May require buffering

### Sampling
For high-traffic systems, collect subset of data:
- **Head-based sampling**: Decide at trace start
- **Tail-based sampling**: Decide after trace completes
- **Probabilistic**: Random percentage

---

## 4.6 Alerting and Incident Response

### Alerting Best Practices

**Good Alert Characteristics**:
- Actionable (someone needs to do something)
- Significant (impacts users or business)
- Clear (describes the problem)
- Avoiding alert fatigue

**Alertmanager (Prometheus)**:
- Groups related alerts
- Silences and inhibitions
- Routing to different receivers
- Notification channels (email, Slack, PagerDuty)

**Example Alert Rule**:
```yaml
groups:
- name: example
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status="500"}[5m]) > 0.1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value }} errors/second"
```

### Service Level Objectives (SLOs)

**Key Terms**:
- **SLI (Service Level Indicator)**: Measurement of service behavior (e.g., latency, error rate)
- **SLO (Service Level Objective)**: Target value for an SLI (e.g., 99.9% availability)
- **SLA (Service Level Agreement)**: Contract with consequences for missing SLO

**Example**:
- SLI: Percentage of requests < 200ms
- SLO: 99.5% of requests < 200ms
- SLA: If SLO missed, customer gets credit

---

## 4.7 Cost Management and Optimization

### Cloud Cost Observability

**Why Monitor Costs?**:
- Cloud resources are pay-per-use
- Easy to overspend
- Identify waste and optimization opportunities

**Cost Metrics to Track**:
- Resource utilization vs allocation
- Idle resources
- Over-provisioned workloads
- Spot instance savings

**Tools for Kubernetes Cost Management**:
| Tool | Description |
|------|-------------|
| **Kubecost** | Kubernetes cost monitoring |
| **AWS Cost Explorer** | AWS-specific cost analysis |
| **Google Cloud Billing** | GCP cost tracking |
| **FinOps** | Financial operations practices |

### Resource Optimization
- Right-size Pod resource requests/limits
- Use VPA recommendations
- Implement cluster autoscaling
- Consider spot/preemptible instances

---

## Key Exam Tips for This Domain

1. **Know the three pillars**: Metrics, Logs, Traces
2. **Understand Prometheus** architecture and PromQL basics
3. **Know OpenTelemetry's** purpose (unified observability standard)
4. **Understand the difference** between Fluentd and Fluent Bit
5. **Know Jaeger's** role in distributed tracing
6. **Understand Golden Signals**: Latency, Traffic, Errors, Saturation
7. **Know SLI/SLO/SLA** definitions
8. **Understand push vs pull** models for metrics collection

---

## Practice Questions

1. What are the three pillars of observability?
   - Answer: Metrics, Logs, and Traces

2. Which CNCF project is used for metrics collection in Kubernetes?
   - Answer: Prometheus

3. What is the purpose of OpenTelemetry?
   - Answer: Provide a unified, vendor-neutral standard for collecting metrics, logs, and traces

4. What query language does Prometheus use?
   - Answer: PromQL (Prometheus Query Language)

5. What are the Google SRE Golden Signals?
   - Answer: Latency, Traffic, Errors, Saturation

6. What is the difference between Fluentd and Fluent Bit?
   - Answer: Fluentd is full-featured (Ruby-based), Fluent Bit is lightweight (C-based)

7. What does a trace represent in distributed tracing?
   - Answer: The complete journey of a request through all services

8. Which tool is used for log aggregation that indexes only labels?
   - Answer: Grafana Loki

---

## Quick Reference: Observability Stack

| Component | Purpose | Tool Examples |
|-----------|---------|---------------|
| **Metrics Collection** | Gather numerical data | Prometheus, OpenTelemetry |
| **Log Collection** | Gather event logs | Fluentd, Fluent Bit, Vector |
| **Log Storage** | Store logs | Loki, Elasticsearch |
| **Tracing** | Track request flow | Jaeger, Zipkin |
| **Visualization** | Dashboards | Grafana |
| **Alerting** | Notify on issues | Alertmanager, PagerDuty |

---

## Additional Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [Grafana Loki Documentation](https://grafana.com/docs/loki/)
- [Google SRE Book - Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)