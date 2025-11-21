## Dynatrace MCP for OpenTelemetry Demo

This guide covers using the Dynatrace MCP server to monitor and troubleshoot the OpenTelemetry Astronomy Shop demo.

### Quick Start

The demo runs in Kubernetes with namespace typically named `otel-demo` or similar. All services export telemetry to the OpenTelemetry Collector, which forwards to Dynatrace.

### Finding Demo Services

```dql
// Find all demo services
find_entity_by_name("otel-demo")

// Locate specific services (adjust namespace as needed)
fetch dt.entity.service
| filter matchesValue(entity.name, "*otel-demo*")
| fields id, entity.name, calls, called_by
```

Service naming pattern: `[<cluster>][otel-demo] <service-name>`

**Core services to monitor:**
- frontend (Next.js)
- product-catalog (Go)
- cart (C#)
- checkout (Go)
- payment (Node.js)
- shipping (Rust)
- recommendation (Python)
- ad (Java)
- email (Ruby)
- accounting (C#)
- fraud-detection (Kotlin)

### Common Investigation Scenarios

#### 1. Service Performance Issues

```dql
// Find slow services in the demo
timeseries avg(dt.service.request.response_time), 
  percentile(dt.service.request.response_time, 95),
  from: now() - 1h, interval: 1m,
  filter: matchesValue(entity.name, "*otel-demo*"),
  by: {dt.entity.service}

// Check specific service (e.g., checkout)
timeseries avg(dt.service.request.response_time),
  sum(dt.service.request.failure_count),
  from: now() - 2h, interval: 5m,
  filter: matchesValue(entity.name, "*checkout*")
```

#### 2. Error Analysis

```dql
// Find which demo services have errors
fetch logs 
| filter matchesValue(k8s.namespace.name, "*otel*") 
  and loglevel == "ERROR" 
  and timestamp > now() - 1h
| summarize errorCount = count(), by:{k8s.deployment.name}
| sort errorCount desc

// Analyze error patterns for a specific service
fetch logs 
| filter contains(k8s.deployment.name, "payment") 
  and loglevel == "ERROR" 
  and timestamp > now() - 1h
| summarize cnt = count(), by:{content}
| sort cnt desc | limit 20
```

#### 3. Resource Usage

```dql
// Container CPU/Memory across all demo services
timeseries avg(dt.kubernetes.container.cpu_usage),
  avg(dt.kubernetes.container.memory_working_set),
  from: now() - 1h, interval: 1m,
  filter: matchesValue(k8s.namespace.name, "*otel*"),
  by: {k8s.deployment.name}

// Check if services are hitting resource limits
fetch dt.entity.cloud_application_instance 
| filter matchesValue(k8s.namespace.name, "*otel*")
| summarize avgCPU = avg(dt.kubernetes.container.cpu_usage),
    cpuLimit = max(dt.kubernetes.container.limits_cpu),
    avgMemory = avg(dt.kubernetes.container.memory_working_set),
    memoryLimit = max(dt.kubernetes.container.limits_memory),
    by:{k8s.deployment.name}
```

#### 4. Service Dependencies

```dql
// Map service call relationships
fetch dt.entity.service
| filter matchesValue(entity.name, "*otel-demo*")
| fields entity.name, calls, called_by

// Trace slow requests through the stack
fetch spans 
| filter matchesValue(k8s.namespace.name, "*otel*")
  and timestamp > now() - 30m 
  and duration > 1000000000
| summarize avgDuration = avg(duration), 
    p95 = percentile(duration, 95), 
    cnt = count(), 
    by:{dt.entity.service, span.name}
| sort p95 desc
```

#### 5. Kafka Message Flow

The demo uses Kafka for checkout, accounting, and fraud-detection services:

```dql
// Check Kafka-related errors
fetch logs
| filter matchesValue(k8s.namespace.name, "*otel*")
  and (contains(content, "kafka") or contains(content, "Kafka"))
  and loglevel == "ERROR"
  and timestamp > now() - 1h
| fields timestamp, k8s.deployment.name, content

// Monitor services consuming from Kafka
fetch logs
| filter (contains(k8s.deployment.name, "accounting") 
    or contains(k8s.deployment.name, "fraud-detection"))
  and timestamp > now() - 30m
| summarize logCount = count(), by:{k8s.deployment.name, loglevel}
```

#### 6. Database Performance

PostgreSQL is used by accounting and product-reviews services:

```dql
// Find database-related errors
fetch logs
| filter matchesValue(k8s.namespace.name, "*otel*")
  and (contains(content, "postgres") or contains(content, "database"))
  and loglevel == "ERROR"
  and timestamp > now() - 1h
| fields timestamp, k8s.deployment.name, content

// Check services with DB connections
fetch logs
| filter (contains(k8s.deployment.name, "accounting")
    or contains(k8s.deployment.name, "product-reviews"))
  and timestamp > now() - 30m
| summarize cnt = count(), by:{k8s.deployment.name, loglevel}
```

#### 7. Frontend Performance

```dql
// Frontend service metrics
timeseries avg(dt.service.request.response_time),
  sum(dt.service.request.count),
  from: now() - 2h, interval: 5m,
  filter: matchesValue(entity.name, "*frontend*")

// Frontend errors and warnings
fetch logs
| filter contains(k8s.deployment.name, "frontend")
  and (loglevel == "ERROR" or loglevel == "WARN")
  and timestamp > now() - 1h
| summarize cnt = count(), by:{loglevel, content}
| sort cnt desc | limit 20
```

### Language-Specific Metrics

#### Java Services (ad, fraud-detection)

```dql
// JVM heap usage
timeseries avg(dt.runtime.jvm.memory.heap.used),
  avg(dt.runtime.jvm.memory.heap.max),
  from: now() - 1h,
  filter: contains(k8s.deployment.name, "ad") 
    or contains(k8s.deployment.name, "fraud-detection"),
  by: {k8s.deployment.name}
```

#### Go Services (checkout, product-catalog)

```dql
// Goroutine and memory metrics
timeseries avg(dt.runtime.go.scheduler.goroutine_count),
  avg(dt.runtime.go.memory.heap),
  from: now() - 1h,
  filter: contains(k8s.deployment.name, "checkout")
    or contains(k8s.deployment.name, "product-catalog"),
  by: {k8s.deployment.name}
```

### Problem Detection

```dql
// Find active problems in the demo
list_problems(filter: "contains(k8s.namespace.name, \"otel-demo\")")

// Problems affecting specific services
list_problems(filter: "in(affected_entity_ids, \"<service-id>\")")

// Get AI insights on problems
chat_with_davis_copilot("What's causing the high error rate in the checkout service?")
```

### Load Generator Analysis

The demo includes a load-generator service (Locust):

```dql
// Monitor load generator activity
fetch logs
| filter contains(k8s.deployment.name, "load-generator")
  and timestamp > now() - 30m
| summarize cnt = count(), by:{loglevel}

// Correlate load with service performance
timeseries avg(dt.service.request.count),
  from: now() - 1h, interval: 1m,
  filter: matchesValue(k8s.namespace.name, "*otel*"),
  by: {dt.entity.service}
```

### Best Practices for Demo Monitoring

1. **Start with namespace filter**: Always filter by `k8s.namespace.name` to isolate demo traffic
2. **Use short time ranges**: Demo generates continuous load, so `now() - 1h` is usually sufficient
3. **Monitor service interactions**: The demo showcases distributed tracing - use span queries to follow requests
4. **Check all log levels**: Demo services log at INFO, WARN, and ERROR levels
5. **Watch resource usage**: Services have memory limits (20M-500M) - monitor for OOMKilled events

### Quick Health Check

```dql
// Overall demo health snapshot
fetch logs
| filter matchesValue(k8s.namespace.name, "*otel*")
  and timestamp > now() - 15m
| summarize logCount = count(), 
    errorCount = countIf(loglevel == "ERROR"),
    warnCount = countIf(loglevel == "WARN"),
    by:{k8s.deployment.name}
| fields k8s.deployment.name, logCount, errorCount, warnCount,
    errorRate = errorCount / logCount * 100
| sort errorRate desc
```

### Troubleshooting Tips

- **Service not found**: Check namespace name - it may be `opentelemetry-demo`, `otel-demo`, or custom
- **No metrics**: Verify OpenTelemetry Collector is forwarding to Dynatrace
- **High cardinality**: Demo generates many trace IDs - always aggregate or filter by service/deployment
- **Missing logs**: Some services are more verbose than others - check container logs directly if needed
