# GitHub Copilot Instructions — Observability‑Driven Development (ODD)

These instructions guide Copilot to support **observability‑driven development**: use real production signals (or best available telemetry) during design and implementation to **avoid breaking the system**, and to ship safely.

## Repo context (don’t guess)

Do not assume what the repo contains.

Before suggesting code changes or observability queries, first discover:

1) **How the system runs**
   - Kubernetes, Docker Compose, or something else?
   - Where are manifests/configs (`kubernetes/`, `docker-compose.yml`, Helm charts, etc.)?

2) **What the services are called**
   - In code: look for service folders, Docker Compose service names, Helm release names, and environment variables.
   - In telemetry: identify the runtime identifiers (Kubernetes namespace, deployment names, service entity names, tags).

3) **What the critical path is**
   - Identify the user-facing entrypoint(s) and the downstream dependencies.

If you are unsure how a service is built/run/instrumented, read that service’s `README.md` before making assumptions.

## ODD value proposition (the why)

- Traditional development is often **reactive**: failures are discovered after deployment.
- Many incidents come from **design assumptions** (latency, timeouts, resource constraints) that don’t match reality.
- ODD uses **real production data + AI during design** to predict risk, prevent failures before coding, and keep delivery fast.

## Default workflow for changes

When asked to implement or change behavior (especially hot paths):

1) **Identify the critical path**
   - Is this code in the request path for high‑traffic operations?
   - Does it add synchronous I/O, new dependencies, or expensive computation?

2) **Check production reality (preferred) or establish a baseline (fallback)**
   - If Dynatrace is available, query metrics/logs/traces before adding synchronous work on hot paths.
    - If Dynatrace is not available in the current environment, state the limitation and:
       - propose the exact DQL you would run (see “Dynatrace queries” below), and/or
       - establish a local baseline (e.g., run the demo + load-generator and compare p95 latency / error rate before vs after).

3) **Risk gate**
   - If the target service is already near saturation (CPU/memory/latency/error spikes), do **not** add complexity.
   - Recommend mitigations first: scale resources, reduce scope, add caching, async patterns, or change the design.

4) **Implement with guardrails**
   - Feature‑flag new behavior when feasible.
   - Add timeouts, retries, and circuit breakers when calling other services.
   - Prefer connection reuse over per‑request client creation.
   - Prefer graceful degradation for non‑critical validation (fail‑open vs fail‑closed must be explicit).

5) **Instrument and verify**
   - Emit traces/metrics/logs needed to evaluate impact (latency, error rate, resource usage).
   - Add tests for correctness and failure modes.
   - Run the narrowest relevant tests/builds.

6) **Rollout + monitoring guidance**
   - Provide a safe rollout plan (gradual enablement, rollback trigger thresholds).
   - Specify what dashboards/queries to watch and what “good” looks like.

## Observability requirements for code changes

When modifying service behavior, Copilot should ensure:

- **Traces:** add spans for new external calls or major sub-operations.
- **Metrics:** record duration histograms and result counters for new logic.
- **Logs:** include structured fields (service, operation, key IDs) at appropriate levels.

If the repo already has established instrumentation patterns in the touched service, follow them.

## Hot-path safety rules

On endpoints and handlers that run for every user interaction:

- Avoid adding synchronous downstream calls unless you’ve validated headroom.
- If a downstream call is required:
  - enforce a bounded timeout,
  - implement retries carefully (small count, only on safe errors),
  - use circuit breaking,
  - ensure telemetry attributes exist to debug failures quickly.

## Dynatrace‑assisted development (if available)

If asked “is this safe for production?”, answer in terms of:
- baseline vs expected deltas (CPU, memory, latency, error rate)
- current saturation signals (spikes, upward trends, throttling, OOM/restarts)
- dependency capacity (downstream services)
- explicit go/no-go criteria and rollback triggers

### Dynatrace queries (DQL examples)

Use short time windows for triage and longer windows for trend verification.

#### Step 0: Discover the right filters (don’t guess)

Use these to learn the real values for `k8s.namespace.name`, `k8s.deployment.name`, and/or `dt.entity.service`.

**Discover active Kubernetes namespaces**
```dql
fetch logs
| filter timestamp > now() - 1h
| summarize logCount = count(), by:{k8s.namespace.name}
| sort logCount desc
```

**Discover active deployments within a namespace**
```dql
fetch logs
| filter timestamp > now() - 1h
   and k8s.namespace.name == "<YOUR_NAMESPACE>"
| summarize logCount = count(), by:{k8s.deployment.name}
| sort logCount desc
```

If service entity names are used instead of k8s fields in your environment, first list services and then narrow:
```dql
fetch dt.entity.service
| fields id, entity.name
| sort entity.name asc
```

**Kubernetes CPU/Memory by deployment**
```dql
timeseries avg(dt.kubernetes.container.cpu_usage),
   avg(dt.kubernetes.container.memory_working_set),
   from: now() - 1h, interval: 1m,
   filter: k8s.namespace.name == "<YOUR_NAMESPACE>",
   by: {k8s.deployment.name}
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

**Service response time + failures (service metrics)**
```dql
timeseries avg(dt.service.request.response_time),
   percentile(dt.service.request.response_time, 95),
   sum(dt.service.request.failure_count),
   from: now() - 1h, interval: 1m,
   // Optional: add a filter for the specific service/entity naming scheme in your environment
   by: {dt.entity.service}
```

**Slow spans (find hot operations)**
```dql
fetch spans
| filter k8s.namespace.name == "<YOUR_NAMESPACE>"
   and timestamp > now() - 30m
   and duration > 1000000000
| summarize p95 = percentile(duration, 95), cnt = count(), by:{dt.entity.service, span.name}
| sort p95 desc
```

## Output expectations from Copilot

For non-trivial changes, Copilot should include in its response:

- What was changed (files + intent)
- What observability was added/updated (traces/metrics/logs)
- What risk checks were performed (or why they couldn’t be)
- How to validate locally (commands/tests)
- What to monitor and what thresholds would trigger rollback

## Scope and style

- Keep changes minimal and consistent with the existing codebase.
- Don’t introduce new frameworks or broad refactors unless explicitly requested.
- Don’t invent new product requirements; if ambiguous, choose the simplest safe interpretation and call out assumptions.
