# GitHub Copilot Instructions — Observability-Driven Development (ODD)

These instructions guide Copilot to support observability-driven development: use real runtime signals, or the best local substitute, during design and implementation to avoid breaking the system and to ship safely.

## Agentic expectations

When the user asks to implement a change:

- Prefer taking action over theorizing: explore the repo, implement the smallest safe change, and validate it.
- Use a short plan for non-trivial work.
- Treat observability as part of the feature: changes are not done until they can be evaluated via traces, metrics, and logs.
- If the request is ambiguous, ask 1-3 precise clarifying questions or state assumptions explicitly.

When you cannot access Dynatrace or `dtctl`, say so and use the local baseline workflow.

## Repo context

Do not assume what the repo contains.

Before suggesting code changes or observability queries, first discover:

1. How the system runs.
   - Kubernetes, Docker Compose, or something else?
   - Where are manifests and configs such as `kubernetes/` or `docker-compose.yml`?
2. What the services are called.
   - In code: service folders, Docker Compose service names, Helm release names, and environment variables.
   - In telemetry: namespace, deployment, service entity names, and tags.
3. What the critical path is.
   - Identify the user-facing entrypoints and the downstream dependencies.

If you are unsure how a service is built, run, or instrumented, read that service's `README.md` before making assumptions.

## ODD value proposition

- ODD reduces incidents caused by mismatched assumptions about latency, timeouts, and capacity.
- Changes are only done when they are measurable and safe to roll back.

## dtctl-first operating model

Prefer `dtctl` as the default interface to Dynatrace.

Why:

- It covers query verification and execution.
- It manages workflows, dashboards, notebooks, SLOs, settings objects, lookups, and buckets.
- It supports contexts, safety levels, dry runs, diffs, and templated apply flows.
- It is agent-friendly through `dtctl commands --brief -o json`, `--plain`, and `--agent`.

Dynatrace MCP is optional and secondary. Use it when it is already available and clearly faster for a narrow task, but do not block ODD on MCP availability.

## Default workflow for changes

When asked to implement or change behavior, especially on hot paths:

1. Identify the critical path.
   - Is this code in the request path for high-traffic operations?
   - Does it add synchronous I/O, new dependencies, or expensive computation?
2. Check production reality or establish a fallback baseline.
   - Start with `dtctl doctor`, `dtctl ctx`, and `dtctl commands --brief -o json` if you need capability discovery.
   - Verify DQL before running it: `dtctl verify query ...`, then `dtctl query ...`.
   - If Dynatrace or `dtctl` is unavailable, state the limitation and either propose the `dtctl` commands you would run or establish a local baseline with the demo and `load-generator`.
3. Apply a risk gate.
   - If the target service is already near saturation, do not add complexity.
   - Recommend mitigations first: narrower scope, caching, async patterns, workflow-based checks, or a different design.
4. Implement with guardrails.
   - Feature-flag new behavior when feasible.
   - Add timeouts, retries, and circuit breakers when calling other services.
   - Prefer connection reuse over per-request client creation.
   - Prefer graceful degradation for non-critical validation.
5. Instrument and verify.
   - Emit traces, metrics, and logs needed to evaluate impact.
   - Add tests for correctness and failure modes.
   - Run the narrowest relevant tests and builds.
   - When the change needs durable operational support, use `dtctl` to create or update workflows, dashboards, notebooks, SLOs, settings objects, lookups, or buckets.
6. Provide rollout and monitoring guidance.
   - Give a safe rollout plan.
   - Specify what to monitor and what good looks like.

## Definition of done

For any non-trivial behavior change, the final output should include:

- correctness validation covering success and failure modes
- explicit safety for new downstream calls
- observability that makes the new logic measurable
- a before/after baseline plan based on `dtctl` queries or a local run plus load
- rollback triggers grounded in error rate, latency, or saturation

## Observability requirements for code changes

When modifying service behavior, ensure:

- traces for new external calls or major sub-operations
- metrics for duration and result counters
- logs with structured fields at appropriate levels

If the repo already has instrumentation conventions in the touched service, follow them.

### Telemetry conventions

- Reuse existing metric naming and prefixes in the service you touch.
- Prefer OpenTelemetry semantic conventions for attributes when they exist.
- For logs, include correlation fields when possible and structured business identifiers that are not sensitive.
- For spans, add actionable attributes but avoid high-cardinality values.

### High-cardinality guardrail

Do not add unbounded or high-cardinality labels such as raw user input, emails, full IDs, or unique URLs to metrics or span attributes.

## Hot-path safety rules

On endpoints and handlers that run for every user interaction:

- Avoid adding synchronous downstream calls unless you have validated headroom.
- If a downstream call is required:
  - enforce a bounded timeout
  - implement retries carefully and only on safe errors
  - use circuit breaking where appropriate
  - ensure telemetry attributes exist to debug failures quickly

## Risk tiers

- Tier 0: docs, refactor, or no behavior change -> minimal validation
- Tier 1: non-hot-path behavior change -> targeted tests plus basic telemetry
- Tier 2: hot-path change, synchronous downstream call, payload change, retry change, or runtime config change -> baseline comparison required, strict budgets required, explicit rollback triggers required

## Dynatrace-assisted development

If asked whether a change is safe for production, answer in terms of:

- baseline versus expected deltas for CPU, memory, latency, and error rate
- current saturation signals such as spikes, throttling, OOMs, or restart loops
- dependency capacity
- explicit go or no-go criteria and rollback triggers

### dtctl workflow

1. Confirm connection and safety context.
   - `dtctl doctor`
   - `dtctl ctx`
   - `dtctl auth whoami` when identity matters
2. Discover entities and artifacts instead of guessing.
   - services, namespaces, deployments, workflows, dashboards, notebooks, SLOs, settings schemas
3. Verify and then execute queries.
   - `dtctl verify query ...`
   - `dtctl query ...`
4. Operationalize the result when useful.
   - `dtctl apply -f workflow.yaml`
   - `dtctl apply -f dashboard.yaml`
   - `dtctl apply -f notebook.yaml`
   - `dtctl apply -f settings.yaml`
   - `dtctl apply -f slo.yaml`
5. Summarize findings as deltas and risk decisions.

Rule of thumb for query cost:

- use short windows for triage
- use longer windows for baselines only with tight filters and aggregation
- filter by time early, then narrow by stable identifiers before text search

### Query hygiene

- Keep each DQL snippet to a single query.
- `summarize` requires an aggregation.
- `sort` must sort by an alias.
- Avoid `in(...)` or set-literal syntax when explicit `or` chains are clearer.
- Do not assume Kubernetes fields exist on entity record types.
- Do not expect entity metadata queries to behave like timeseries metric queries.

### When reviewing or fixing a DQL query

- Identify syntax, field, and filter issues before execution.
- Fix and verify first with `dtctl verify query`, then execute with `dtctl query`.
- If a query returns `0 records`, propose the next smallest checks: widen time range, validate field existence, check filter logic, validate entity IDs.

### dtctl control-plane usage

Use `dtctl` for more than queries when the change warrants it:

- workflows: `get`, `describe`, `edit`, `apply`, `exec`, `logs`
- dashboards and notebooks: `get`, `describe`, `edit`, `apply`, `share`
- SLOs: `get`, `create`, `apply`, `exec`
- settings and OpenPipeline config: `get settings-schemas`, `get`, `create`, `update`, `apply settings`
- lookups and buckets for enrichment and storage changes

Prefer `apply`, `--dry-run`, `diff`, and safer contexts for repeatable, low-risk changes.

## Local baseline workflow

When Dynatrace or `dtctl` is unavailable, establish a before/after baseline locally:

- use the repo's documented run path through Docker Compose or Kubernetes
- use the included `load-generator` to create consistent traffic
- compare error rate and p95 latency before and after for the relevant user-facing entrypoint
- run the narrowest relevant service tests or builds

## Output expectations from Copilot

For non-trivial changes, include:

- what changed and why
- what observability was added or updated
- what Dynatrace assets were added or updated through `dtctl` when applicable
- what risk checks were performed, or why they could not be performed
- how to validate locally
- what to monitor and what thresholds should trigger rollback

If the work is Tier 2, include a short go or no-go decision with explicit criteria.

## Scope and style

- Keep changes minimal and consistent with the existing codebase.
- Do not introduce new frameworks or broad refactors unless explicitly requested.
- Do not invent new product requirements. If ambiguous, choose the simplest safe interpretation and call out assumptions.
