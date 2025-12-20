# GitHub Copilot Instructions — Observability‑Driven Development (ODD)

These instructions guide Copilot to support **observability‑driven development**: use real production signals (or best available telemetry) during design and implementation to **avoid breaking the system**, and to ship safely.

## Agentic expectations (how Copilot should behave)

When the user asks to implement a change:

- Prefer taking action over theorizing: explore the repo, implement the smallest safe change, and validate it.
- Use a short plan for non-trivial work (multi-file, multi-step, or behavior changes on request paths).
- Treat observability as part of the feature: changes are not “done” until they can be evaluated via traces/metrics/logs.
- If the request is ambiguous, ask 1–3 precise clarifying questions or state assumptions explicitly.

When you cannot access production telemetry (no Dynatrace access), say so and use the local baseline workflow.

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

- ODD reduces incidents caused by mismatched assumptions (latency, timeouts, resource limits).
- Changes are only “done” when they’re measurable via telemetry.

## Default workflow for changes

When asked to implement or change behavior (especially hot paths):

1) **Identify the critical path**
   - Is this code in the request path for high‑traffic operations?
   - Does it add synchronous I/O, new dependencies, or expensive computation?

2) **Check production reality (preferred) or establish a baseline (fallback)**
   - If Dynatrace is available, query metrics/logs/traces before adding synchronous work on hot paths.
    - If Dynatrace is not available in the current environment, state the limitation and:
       - propose the DQL you would run (see `.github/odd-dynatrace-dql.md`), and/or
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

## Definition of Done (ODD)

For any non-trivial behavior change, the final output should include:

- **Correctness:** tests (or at least a minimal reproducible validation) cover success + failure modes.
- **Safety:** timeouts/budgets for new downstream calls; graceful degradation is explicit.
- **Observability:** new logic is traceable and measurable (spans + metrics + logs as appropriate).
- **Verification:** a before/after baseline plan (Dynatrace query set or local run + load).
- **Rollback:** clear trigger thresholds (e.g., error rate, p95 latency, CPU saturation).

## Observability requirements for code changes

When modifying service behavior, Copilot should ensure:

- **Traces:** add spans for new external calls or major sub-operations.
- **Metrics:** record duration histograms and result counters for new logic.
- **Logs:** include structured fields (service, operation, key IDs) at appropriate levels.

If the repo already has established instrumentation patterns in the touched service, follow them.

### Telemetry conventions (prefer existing patterns)

- Reuse existing metric naming and prefixes in the service you touch (do not invent a new scheme).
- Prefer OpenTelemetry semantic conventions for attributes when they exist.
- For logs, include correlation fields when possible (trace/span ids) and add structured fields for key business identifiers (non-sensitive).
- For spans, add attributes that make troubleshooting actionable (peer/service, operation, result, error type), but avoid high-cardinality values.

### High-cardinality guardrail

Do not add unbounded/high-cardinality labels (e.g., raw user input, emails, full IDs, URLs with unique path segments) to metrics or span attributes.

## Hot-path safety rules

On endpoints and handlers that run for every user interaction:

- Avoid adding synchronous downstream calls unless you’ve validated headroom.
- If a downstream call is required:
  - enforce a bounded timeout,
  - implement retries carefully (small count, only on safe errors),
  - use circuit breaking,
  - ensure telemetry attributes exist to debug failures quickly.

## Risk tiers (how much rigor to apply)

- **Tier 0 (docs/refactor/no behavior change):** minimal validation.
- **Tier 1 (non-hot-path behavior change):** targeted tests + basic telemetry.
- **Tier 2 (hot-path / synchronous downstream call / payload or retry changes):** baseline comparison required (Dynatrace preferred), strict timeouts/budgets, and explicit rollback triggers.

## Dynatrace‑assisted development (if available)

If asked “is this safe for production?”, answer in terms of:
- baseline vs expected deltas (CPU, memory, latency, error rate)
- current saturation signals (spikes, upward trends, throttling, OOM/restarts)
- dependency capacity (downstream services)
- explicit go/no-go criteria and rollback triggers

### Dynatrace workflow (recommended)

1) **Confirm connection** (if supported by your environment): get tenant/env info.
2) **Discover entities/filters**: list namespaces/deployments or service entities instead of guessing.
3) **Generate queries**: use short windows for triage and longer windows for trend.
4) **Verify then execute**: syntactically validate DQL before running if needed.
5) **Summarize as deltas**: what changed, expected impact, and go/no-go thresholds.

### Dynatrace queries (DQL templates)

See `.github/odd-dynatrace-dql.md` for:

- discovering namespaces/deployments/services (don’t guess filters)
- CPU/memory by deployment
- error logs by deployment
- service response time/failures
- slow spans / hot operations

## Local baseline workflow (when production telemetry isn’t available)

When Dynatrace is unavailable, establish a before/after baseline locally:

- Use the repo’s documented run path (root `docker compose up` or Kubernetes manifests).
- Use the included load generator (if available in the chosen deployment) to create consistent traffic.
- Compare **error rate** and **p95 latency** before vs after for the most relevant user-facing entrypoint.
- If the change affects a single service, run that service’s narrowest test/build command (see the service `README.md`).

## Output expectations from Copilot

For non-trivial changes, Copilot should include in its response:

- What was changed (files + intent)
- What observability was added/updated (traces/metrics/logs)
- What risk checks were performed (or why they couldn’t be)
- How to validate locally (commands/tests)
- What to monitor and what thresholds would trigger rollback

If the work is Tier 2, include a short “Go/No-Go” decision with explicit criteria.

## Scope and style

- Keep changes minimal and consistent with the existing codebase.
- Don’t introduce new frameworks or broad refactors unless explicitly requested.
- Don’t invent new product requirements; if ambiguous, choose the simplest safe interpretation and call out assumptions.
