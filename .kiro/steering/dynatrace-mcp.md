## Dynatrace MCP for Microservices Apps (Template)

This guide covers using the Dynatrace MCP server to monitor and troubleshoot a microservices-based system.

## Observability-Driven Development (ODD) usage

Use Dynatrace not only for troubleshooting, but also as a **validation gate** for code changes.

### When implementing a behavior change

- **Baseline first (Tier 2):** before adding synchronous work on hot paths (new downstream calls, retries, payload changes), capture a baseline for p95 latency, error rate, and resource saturation signals.
- **After-change verification:** run the same queries after the change and summarize deltas.
- **Rollback criteria:** state explicit thresholds that trigger rollback (e.g., error rate increase, p95 regression, CPU saturation, OOM/restarts).

### Guardrails

- Don’t guess filters: always discover namespace/deployment/service identifiers first.
- Prefer stable identifiers and exact matches over fuzzy patterns when possible.
- Avoid high-cardinality fields/filters in dashboards/metrics strategy (don’t build approaches around raw user input, full IDs, or unique URLs).

### Quick Start

Do not assume runtime details (namespace, cluster, deployment names, service entity naming). Always discover them first.

If your system runs in Kubernetes, most filters will use `k8s.namespace.name` and `k8s.deployment.name`.
If it runs outside Kubernetes, prefer `dt.entity.service`, `service.name`, `cloud.application`, and/or tags available in your environment.

All example queries below are templates. Replace placeholders like `<YOUR_NAMESPACE>` or `<SERVICE_SUBSTRING>` with values discovered from the environment.

### Step 0: Discovery (derive the right filters)

Start broad, then narrow. The goal is to answer:
1) What namespaces and deployments are active?
2) What are the service entity names in Dynatrace?
3) What tag(s) or naming convention can reliably isolate “this system” from everything else?

**Discover active namespaces (Kubernetes environments)**
```dql
fetch logs
| filter timestamp > now() - 1h
| summarize logCount = count(), by:{k8s.namespace.name}
| sort logCount desc
```

**Discover deployments within a namespace**
```dql
fetch logs
| filter timestamp > now() - 1h
  and k8s.namespace.name == "<YOUR_NAMESPACE>"
| summarize logCount = count(), by:{k8s.deployment.name}
| sort logCount desc
```

**Discover services (service entity names)**
```dql
fetch dt.entity.service
| fields id, entity.name
| sort entity.name asc
```

Tip: Once you see stable identifiers, prefer exact matches (`==`) over fuzzy patterns (`contains`/`matchesValue`) to avoid accidental cross-system noise.

### Finding Services (generic)

If you know a likely substring for the system (e.g., a namespace, environment name, or app name), start with that:

```dql
fetch dt.entity.service
| filter contains(entity.name, "<SERVICE_SUBSTRING>")
| fields id, entity.name, calls, called_by
```

If you don’t know any substring, use the discovery queries above first.

### Common Investigation Scenarios

#### 1. Service Performance Issues

```dql
// Find slow services in the system
timeseries avg(dt.service.request.response_time), 
  percentile(dt.service.request.response_time, 95),
  from: now() - 1h, interval: 1m,
  // Optional: add an environment-specific filter once discovered
  by: {dt.entity.service}

// Check specific service (e.g., checkout)
timeseries avg(dt.service.request.response_time),
  sum(dt.service.request.failure_count),
  from: now() - 2h, interval: 5m,
  filter: matchesValue(entity.name, "*checkout*")
```

#### 2. Error Analysis

```dql
// Find which services have errors (Kubernetes)
fetch logs 
| filter k8s.namespace.name == "<YOUR_NAMESPACE>" 
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
// Container CPU/Memory across deployments (Kubernetes)
timeseries avg(dt.kubernetes.container.cpu_usage),
  avg(dt.kubernetes.container.memory_working_set),
  from: now() - 1h, interval: 1m,
  filter: k8s.namespace.name == "<YOUR_NAMESPACE>",
  by: {k8s.deployment.name}

// Check if services are hitting resource limits
fetch dt.entity.cloud_application_instance 
| filter k8s.namespace.name == "<YOUR_NAMESPACE>"
| summarize avgCPU = avg(dt.kubernetes.container.cpu_usage),
    cpuLimit = max(dt.kubernetes.container.limits_cpu),
    avgMemory = avg(dt.kubernetes.container.memory_working_set),
    memoryLimit = max(dt.kubernetes.container.limits_memory),
    by:{k8s.deployment.name}
```

#### 4. Service Dependencies

```dql
// Map service call relationships (narrow with a discovered filter)
fetch dt.entity.service
| filter contains(entity.name, "<SERVICE_SUBSTRING>")
| fields entity.name, calls, called_by

// Trace slow requests through the stack (Kubernetes)
fetch spans 
| filter k8s.namespace.name == "<YOUR_NAMESPACE>"
  and timestamp > now() - 30m 
  and duration > 1000000000
| summarize avgDuration = avg(duration), 
    p95 = percentile(duration, 95), 
    cnt = count(), 
    by:{dt.entity.service, span.name}
| sort p95 desc
```

#### 5. Kafka Message Flow

If your system uses Kafka, you can start with these templates:

```dql
// Check Kafka-related errors
fetch logs
| filter k8s.namespace.name == "<YOUR_NAMESPACE>"
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

If your system uses a database (e.g., PostgreSQL), you can start with these templates:

```dql
// Find database-related errors
fetch logs
| filter k8s.namespace.name == "<YOUR_NAMESPACE>"
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
// Find active problems (narrow with a discovered filter)
list_problems(filter: "k8s.namespace.name == \"<YOUR_NAMESPACE>\"")

// Problems affecting specific services
list_problems(filter: "in(affected_entity_ids, \"<service-id>\")")

// Get AI insights on problems
chat_with_davis_copilot("What's causing the high error rate in the checkout service?")
```

### Load Generator Analysis

If your system includes a load generator, you can correlate generated load with service performance:

```dql
// Monitor load generator activity
fetch logs
| filter contains(k8s.deployment.name, "load-generator")
  and timestamp > now() - 30m
| summarize cnt = count(), by:{loglevel}

// Correlate load with service performance
timeseries avg(dt.service.request.count),
  from: now() - 1h, interval: 1m,
  // Optional: add a filter once you discovered how to isolate your system
  by: {dt.entity.service}
```

### Best Practices

1. **Start with an isolation filter**: Prefer `k8s.namespace.name == "<YOUR_NAMESPACE>"` (Kubernetes) or a stable service naming/tag filter.
2. **Use short time ranges for triage**: `now() - 30m` to `now() - 2h` is usually enough.
3. **Monitor service interactions**: Use span queries to follow requests through the dependency graph.
4. **Check all log levels**: Demo services log at INFO, WARN, and ERROR levels
5. **Watch resource usage**: Monitor CPU/memory and restarts/OOMKilled events.

### Quick Health Check

```dql
// Overall health snapshot (Kubernetes)
fetch logs
| filter k8s.namespace.name == "<YOUR_NAMESPACE>"
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

- **Hook/query returns no data**: Your filter is likely wrong. Re-run the discovery queries and adjust.
- **No metrics**: Verify telemetry export pipeline is configured (OpenTelemetry Collector / OneAgent / ingestion).
- **High cardinality / noisy results**: Narrow filters, aggregate (`summarize`) early, and prefer exact matches.
- **Missing logs**: Some services are more verbose than others - check container logs directly if needed

---

## Production Readiness Analysis for Design Documents

**CRITICAL REQUIREMENT**: When creating or reviewing design documents for features that involve service-to-service communication or performance-sensitive changes, you MUST perform a Dynatrace production readiness analysis.

### When to Perform Analysis

Perform Dynatrace analysis for designs that involve:
- Adding new service-to-service calls (synchronous or asynchronous)
- Modifying existing service call patterns
- Adding timeouts, circuit breakers, or retry logic
- Changes that may impact service latency or throughput
- Features that depend on external service performance

### Required Analysis Steps

#### 1. Identify Affected Services

From the requirements and design documents, identify:
- Which services will be called
- Which services will make new calls
- Expected call frequency and patterns
- Timeout and retry parameters

#### 2. Gather Current Performance Metrics

For each affected service, collect from Dynatrace:

**Latency Distribution** (Last 24-48 hours):
```dql
timeseries avg(dt.service.request.response_time),
  percentile(dt.service.request.response_time, 50),
  percentile(dt.service.request.response_time, 90),
  percentile(dt.service.request.response_time, 95),
  percentile(dt.service.request.response_time, 99),
  from: now() - 24h, interval: 1h,
  filter: dt.entity.service == "<SERVICE-ID>",
  by: {dt.entity.service}
```

**Request Volume and Error Rate**:
```dql
timeseries sum(dt.service.request.count),
  sum(dt.service.request.failure_count),
  from: now() - 24h, interval: 1h,
  filter: dt.entity.service == "<SERVICE-ID>"
```

**Resource Utilization**:
```dql
timeseries avg(dt.kubernetes.container.cpu_usage),
  avg(dt.kubernetes.container.memory_working_set),
  from: now() - 24h, interval: 1h,
  filter: contains(k8s.deployment.name, "<service-name>")
```

**Service Dependencies**:
```dql
fetch dt.entity.service
| filter id == "<SERVICE-ID>"
| fields entity.name, calls, called_by
```

#### 3. Analyze Compatibility with Design

Compare design assumptions against production reality:

| Design Parameter | Production Reality | Risk Level |
|------------------|-------------------|------------|
| Expected latency | Actual P50/P95/P99 | HIGH if mismatch |
| Timeout setting | Service P95/P99 | HIGH if timeout < P95 |
| Expected load | Current request rate | MEDIUM if >30% increase |
| Circuit breaker threshold | Current error rate | MEDIUM if too sensitive |
| Resource capacity | Current CPU/Memory | MEDIUM if near limits |

#### 4. Calculate Impact

**Timeout Rate Estimation**:
- If timeout < P95: ~(100 - 95)% = 5% timeout rate
- If timeout < P90: ~(100 - 90)% = 10% timeout rate
- If timeout < P50: ~50% timeout rate (CRITICAL)

**Load Impact**:
- New calls per hour = (calling service requests/hour)
- Load increase % = (new calls / current calls) × 100
- If >30% increase: HIGH RISK - may degrade performance

**Latency Impact**:
- New latency = baseline + (dependency avg latency)
- If new latency > target SLO: HIGH RISK

#### 5. Risk Assessment

Classify the change as:

**LOW RISK** ✅:
- Timeout > P99 of dependency
- Load increase <10%
- Fail-open pattern implemented
- Circuit breaker configured appropriately
- Target SLOs achievable

**MEDIUM RISK** ⚠️:
- Timeout between P95 and P99
- Load increase 10-30%
- Resilience patterns in place
- May require monitoring and tuning

**HIGH RISK** ❌:
- Timeout < P95 of dependency
- Load increase >30%
- No resilience patterns
- Target SLOs not achievable
- Time-based performance variations

#### 6. Required Actions Based on Risk

**For HIGH RISK changes**:
1. **DO NOT PROCEED** with current design
2. **OPTIMIZE** the dependency service first
3. **ADJUST** design parameters (timeout, SLOs, etc.)
4. **IMPLEMENT** alternative approach (caching, async, etc.)
5. **DOCUMENT** findings and recommendations
6. **RE-ANALYZE** after changes

**For MEDIUM RISK changes**:
1. **ADJUST** design parameters if needed
2. **ADD** additional monitoring and alerting
3. **PLAN** gradual rollout (1% → 10% → 50% → 100%)
4. **PREPARE** rollback procedures
5. **DOCUMENT** monitoring plan

**For LOW RISK changes**:
1. **PROCEED** with design as planned
2. **IMPLEMENT** standard monitoring
3. **DOCUMENT** baseline metrics for comparison

### Documentation Requirements

Create the following documents in the spec directory:

1. **PRODUCTION-READINESS.md**:
   - Current performance metrics
   - Risk assessment
   - Impact analysis
   - Recommendations

2. **LATENCY-INVESTIGATION.md** (if issues found):
   - Root cause analysis
   - Time-based patterns
   - Optimization recommendations

3. **DIAGNOSTIC-QUERIES.md** (if issues found):
   - Ready-to-run DQL queries
   - Investigation checklist
   - Monitoring dashboard setup

### Example Analysis Output

```markdown
## Production Readiness Assessment

**Feature**: Cart Inventory Validation
**Risk Level**: ❌ HIGH RISK

### Current Metrics
- Product Catalog P95: 1300ms
- Design Timeout: 500ms
- Expected Timeout Rate: 40%

### Findings
- Timeout setting incompatible with production performance
- Circuit breaker will open continuously
- Feature will only work 4% of the time

### Recommendations
1. Optimize Product Catalog service (reduce P95 to <400ms)
2. OR increase timeout to 1500ms (accept higher cart latency)
3. OR implement caching layer (bypass dependency)

### Decision
DO NOT PROCEED until Product Catalog P95 is optimized.
```

### Integration with Spec Workflow

**During Design Phase**:
1. Complete initial design document
2. **STOP before finalizing**
3. Perform Dynatrace production readiness analysis
4. Update design based on findings
5. Document risk assessment
6. Get user approval with risk disclosure

**Before Implementation**:
1. Verify production metrics haven't changed
2. Confirm risk assessment is still valid
3. Ensure monitoring is in place

### Davis CoPilot Integration

Use Davis CoPilot to get AI-powered insights:

```
I'm planning to add [describe change]. Based on current metrics:
- Service A: P95 = Xms, load = Y req/hour
- Service B: P95 = Xms, load = Y req/hour
- Proposed timeout: Xms

What risks do you see? Is Service B ready for the additional load?
```

### Continuous Monitoring

After deployment:
- Monitor actual vs. predicted metrics
- Update risk assessment if patterns change
- Feed learnings back into future analyses

---

## Summary

**Always perform Dynatrace analysis before finalizing designs that involve service dependencies.**

This prevents:
- ❌ Deploying incompatible timeout settings
- ❌ Overloading downstream services
- ❌ Missing performance bottlenecks
- ❌ Violating SLOs
- ❌ Creating cascading failures

This ensures:
- ✅ Data-driven design decisions
- ✅ Realistic performance expectations
- ✅ Appropriate resilience patterns
- ✅ Successful production deployments
- ✅ Maintainable systems
