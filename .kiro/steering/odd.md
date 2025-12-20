# Observability-Driven Development (ODD) — Steering

ODD is a delivery constraint: behavior changes are not “done” unless they can be validated via telemetry (traces/metrics/logs) and rolled out safely.

## Default workflow

1) **Identify the critical path**
   - Is the change on a hot path (runs for every user action) or in a background path?

2) **Baseline first (before change)**
   - **Dynatrace-first (default):** capture latency (p95), error rate, and saturation signals (CPU/memory/restarts) for the affected services.
   - If Dynatrace is not available in the current environment, use the local baseline workflow below.

3) **Implement with guardrails**
   - Bound new downstream calls with timeouts.
   - Keep retries small and only on safe errors.
   - Prefer graceful degradation for non-critical validation (explicit fail-open vs fail-closed).

4) **Instrument and verify**
   - Add spans for new sub-operations / downstream calls.
   - Add metrics for duration + success/failure counters.
   - Add structured logs for key operations (avoid sensitive/high-cardinality values).

5) **After-change verification**
   - Re-run the same baseline queries and summarize deltas.
   - State rollback criteria before rollout.

## Risk tiers

- **Tier 0:** docs/refactor/no behavior change → minimal validation.
- **Tier 1:** non-hot-path behavior change → targeted tests + basic telemetry.
- **Tier 2:** hot-path change / sync downstream call / payload or retry change → baseline comparison required, strict timeouts/budgets, explicit rollback triggers.

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
