# Observability-Driven Development (ODD) — Steering

ODD is a delivery constraint: behavior changes are not “done” unless they can be validated via telemetry (traces/metrics/logs) and rolled out safely.

## Default workflow

1) **Identify the critical path**
   - Is the change on a hot path (runs for every user action) or in a background path?
   - What services/components are affected?
   - What are the service dependencies and their current performance characteristics?

2) **Baseline first (before change)**
   - **Dynatrace-first (default):** capture latency (p50/p95/p99), error rate, and saturation signals (CPU/memory/restarts) for the affected services.
   - If Dynatrace is not available in the current environment, use the local baseline workflow below.
   - Capture 24-48 hours of data to understand patterns and variability.
   - Document service dependencies and their performance characteristics.

3) **Implement with guardrails**
   - Bound new downstream calls with timeouts (set > P95 of dependency).
   - Keep retries small and only on safe errors (network, not business logic).
   - Prefer graceful degradation for non-critical validation (explicit fail-open vs fail-closed).
   - Add circuit breakers for external dependencies.
   - Implement bulkheads to isolate failure domains.

4) **Instrument and verify**
   - Add spans for new sub-operations / downstream calls.
   - Add metrics for duration + success/failure counters.
   - Add structured logs for key operations (avoid sensitive/high-cardinality values).
   - Tag telemetry with stable identifiers (service name, operation type, not request IDs).
   - Include correlation IDs for distributed tracing.

5) **After-change verification**
   - Re-run the same baseline queries and summarize deltas.
   - State rollback criteria before rollout.
   - Monitor actual vs. predicted behavior.
   - Update risk assessment if patterns change.

## Risk tiers

- **Tier 0:** docs/refactor/no behavior change → minimal validation.
- **Tier 1:** non-hot-path behavior change → targeted tests + basic telemetry.
- **Tier 2:** hot-path change / sync downstream call / payload or retry change → baseline comparison required, strict timeouts/budgets, explicit rollback triggers.

## Spec Workflow Integration

**⚠️ CRITICAL: Production Readiness Gate for Design Documents**

When creating design documents as part of the spec workflow, you MUST perform a production readiness analysis BEFORE asking the user to approve the design if ANY of these conditions are true:

- ✅ Adding new service-to-service calls (synchronous or asynchronous)
- ✅ Modifying existing service call patterns
- ✅ Adding timeouts, circuit breakers, or retry logic
- ✅ Changes that may impact service latency or throughput
- ✅ Features that depend on external service performance

**Workflow Integration**:
1. Write design document sections (Overview through Data Models)
2. **STOP - Check if production readiness analysis is required** (see conditions above)
3. If required: Perform production readiness analysis (see section below)
4. If required: Add "Production Readiness Assessment" section to design document
5. Continue with Correctness Properties section
6. Ask user to approve design

**Quick Decision Tree**:
```
Does your design add/modify service-to-service calls?
├─ YES → PERFORM ANALYSIS (mandatory)
└─ NO → Does it add timeouts/retries/circuit breakers?
    ├─ YES → PERFORM ANALYSIS (mandatory)
    └─ NO → Does it impact latency/throughput?
        ├─ YES → PERFORM ANALYSIS (mandatory)
        └─ NO → Analysis optional
```

## Production Readiness Analysis Framework

### Required Analysis Steps

#### 1. Identify Affected Services
From the requirements and design documents, identify:
- Which services will be called
- Which services will make new calls
- Expected call frequency and patterns
- Timeout and retry parameters

#### 2. Gather Current Performance Metrics
For each affected service, collect baseline metrics:
- **Latency Distribution**: P50, P90, P95, P99 over 24-48 hours
- **Request Volume**: Current requests per minute/hour
- **Error Rate**: Current failure percentage
- **Resource Utilization**: CPU, memory, connection pools
- **Service Dependencies**: What services does it call, what calls it

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
- If timeout < P95: ~5% timeout rate
- If timeout < P90: ~10% timeout rate  
- If timeout < P50: ~50% timeout rate (CRITICAL)

**Load Impact**:
- New calls per hour = (calling service requests/hour)
- Load increase % = (new calls / current calls) × 100
- If >30% increase: HIGH RISK - may degrade performance

**Latency Impact**:
- New latency = baseline + (dependency avg latency)
- If new latency > target SLO: HIGH RISK

#### 5. Risk Assessment

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

### Documentation Template for Design Documents

Add this section to design documents when production readiness analysis is required:

```markdown
## Production Readiness Assessment

### Affected Services
- **Calling Service**: [Service Name] ([Language])
- **Dependency Service**: [Service Name] ([Language])
- **Call Pattern**: [Synchronous/Async/gRPC/HTTP/etc.]
- **Expected Frequency**: [X calls/min]

### Current Production Metrics (Last 24-48h)
**[Dependency Service Name]**:
- P50 Latency: [X]ms
- P95 Latency: [X]ms  
- P99 Latency: [X]ms
- Request Rate: [X] req/min
- Error Rate: [X]%
- CPU Usage: [X]%
- Memory Usage: [X]%

### Design Parameters vs Production Reality
| Parameter | Design Value | Production Reality | Compatible? |
|-----------|-------------|-------------------|-------------|
| Timeout | [X]ms | P95: [X]ms, P99: [X]ms | ✅/⚠️/❌ |
| Expected Load | +[X]% | Current: [X] req/min | ✅/⚠️/❌ |
| Error Handling | [Strategy] | Current error rate: [X]% | ✅/⚠️/❌ |

### Risk Assessment
**Overall Risk Level**: ✅ LOW / ⚠️ MEDIUM / ❌ HIGH

**Key Findings**:
- [Finding 1]
- [Finding 2]
- [Finding 3]

### Decision
[PROCEED / PROCEED WITH MODIFICATIONS / DO NOT PROCEED]
[Explanation of decision and any required changes]
```

## Guardrails

- **High-cardinality safety:** don’t put raw user input, full IDs, emails, or unique URLs into metric labels or span attributes.
- **Prefer stable identifiers:** use canonical entity IDs, namespaces, and deployment names for scoping.
- **Don’t guess production reality:** if you can’t measure headroom, do not add synchronous work on hot paths.

## Dynatrace (default)

Dynatrace-first applies here: use Dynatrace as the primary validation gate for behavior changes.

If Dynatrace is not available in the current environment, fall back to the local baseline workflow.

- Discover the right filters (namespace/deployment/service entity) before narrowing.
- Verify queries before executing.
- Keep each DQL example to a single query; avoid multi-statement blocks.

### Minimal DQL templates (copy/paste)

**Discover active Kubernetes namespaces**

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

**List service entities (entity names)**

```dql
fetch dt.entity.service
| fields id, entity.name
| sort entity.name asc
```

**Errors in logs by deployment**

```dql
fetch logs
| filter k8s.namespace.name == "<YOUR_NAMESPACE>"
   and loglevel == "ERROR"
   and timestamp > now() - 1h
| summarize errorCount = count(), by:{k8s.deployment.name}
| sort errorCount desc
```

**CPU/memory by deployment**

```dql
timeseries avg(dt.kubernetes.container.cpu_usage),
   avg(dt.kubernetes.container.memory_working_set),
   from: now() - 1h, interval: 1m,
   filter: k8s.namespace.name == "<YOUR_NAMESPACE>",
   by: {k8s.deployment.name}
```

**Service latency + failures (service metrics)**

```dql
timeseries avg(dt.service.request.response_time),
   percentile(dt.service.request.response_time, 95),
   sum(dt.service.request.failure_count),
   from: now() - 1h, interval: 1m,
   by: {dt.entity.service}
```

**Slow spans (hot operations)**

```dql
fetch spans
| filter k8s.namespace.name == "<YOUR_NAMESPACE>"
   and timestamp > now() - 30m
   and duration > 1000000000
| summarize p95 = percentile(duration, 95), cnt = count(), by:{dt.entity.service, span.name}
| sort p95 desc
```

**Metric discovery (when you don’t know the key)**

```dql
fetch metric.series
| filter dt.entity.service == "<SERVICE-ID>"
| summarize cnt = count(), by:{metric.key}
| sort cnt desc | limit 50
```

### DQL hygiene (common failure modes)

- `summarize` must include an aggregation function.
- `sort` must sort by an alias (not by `count()` / `avg(...)` directly).
- Avoid `in(...)` / set-literal syntax; use explicit `or` chains.
- Don’t assume K8s fields exist on entity record types (e.g., `fetch dt.entity.service`).
- **Entity metadata vs metrics:** entity record types (`fetch dt.entity.*`) return metadata; use `timeseries ...` for metrics (don’t try to select metric keys as entity fields).

### Time ranges & query cost (rule of thumb)

- Prefer `now() - 30m` to `now() - 2h` for active triage.
- Use `now() - 24h` for baselines, but keep aggregation (`summarize`/`timeseries`) and filters tight.
- Filter by time early, then narrow by stable identifiers (entity IDs, namespace/deployment), then do text search.

### Agentic query behavior

- Execute the narrowest relevant query when asked for data; show results first, then interpret.
- Never execute a query you already know is broken; fix/verify first.
- If results are empty, state “0 records” and propose 1–3 minimal next checks (time range, field existence, filter logic).

### “Zero results” checklist

1. Widen time range.
2. Validate field existence via a minimal `fetch <type> | limit 5`.
3. Re-check case/values and `and` logic.
4. Validate canonical entity IDs (e.g., `SERVICE-...`).

## Local baseline (when production telemetry isn’t available)

- Run the demo with consistent traffic (load generator if available).
- Compare before/after error rate and p95 latency for the user-facing entrypoint.
- Keep the validation narrow to the affected services.

## Observability Best Practices

### Telemetry Design Patterns

**Service Interaction Patterns**:
- Always trace service-to-service calls with correlation IDs
- Include timeout and retry configuration in span attributes
- Log circuit breaker state changes
- Track connection pool metrics for database/external services

**Error Handling Patterns**:
- Distinguish between retryable and non-retryable errors
- Log error context without sensitive data
- Use structured logging with consistent field names
- Include error rates in service health metrics

**Performance Monitoring**:
- Track P50, P95, P99 latencies for all operations
- Monitor resource utilization trends (CPU, memory, connections)
- Set up alerts for SLO violations
- Use distributed tracing to identify bottlenecks

### Common Investigation Scenarios

**Service Performance Issues**:
1. Check service latency distribution over time
2. Identify slow operations using span analysis
3. Correlate with resource utilization metrics
4. Examine service dependencies and their health

**Error Analysis**:
1. Aggregate errors by service and error type
2. Look for patterns in error timing and frequency
3. Check if errors correlate with deployments or load changes
4. Trace error propagation through service dependencies

**Resource Utilization**:
1. Monitor CPU, memory, and connection pool usage
2. Check for resource limits and throttling
3. Identify services approaching capacity limits
4. Correlate resource usage with request volume

### Troubleshooting Checklist

**When investigating issues**:
- [ ] Start with the user-facing service and work backwards
- [ ] Check recent deployments and configuration changes
- [ ] Verify service dependencies are healthy
- [ ] Look for patterns in timing (time of day, day of week)
- [ ] Check for resource constraints (CPU, memory, connections)
- [ ] Examine error logs for root cause indicators
- [ ] Validate monitoring and alerting coverage

**When results are unexpected**:
- [ ] Verify time ranges and filters are correct
- [ ] Check for data sampling or aggregation effects
- [ ] Validate service and deployment names
- [ ] Confirm telemetry is being collected properly
- [ ] Look for configuration drift or environment differences

## Summary

**Key Takeaways**:
- Always measure before changing service interactions
- Design for graceful degradation and failure isolation
- Use production data to validate design assumptions
- Implement comprehensive monitoring and alerting
- Plan rollback procedures before deployment