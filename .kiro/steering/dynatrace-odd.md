# Observability-Driven Development (ODD) with Dynatrace and dtctl

ODD is a delivery constraint: behavior changes are not done until they can be measured, validated, and rolled back safely.

For this repo, treat Dynatrace as both:

- the runtime truth source for logs, metrics, spans, events, and service health
- the control plane for operational artifacts such as workflows, dashboards, notebooks, SLOs, settings objects, lookups, and buckets

## Operating model

Prefer `dtctl` as the default interface to Dynatrace.

Why:

- It covers both read paths and write paths.
- It is agent-friendly: `dtctl commands --brief -o json`, `--plain`, and `--agent` make it scriptable and self-describing.
- It supports environment contexts, safety levels, dry runs, diffs, verification, and templated apply flows.
- It lets the same ODD workflow move from discovery to automation without switching tools.

Dynatrace MCP is optional and secondary. Use it only when it is already available and materially faster for the task. Do not block ODD on MCP availability.

If `dtctl` or Dynatrace access is unavailable, say so explicitly and use the local baseline workflow.

## dtctl bootstrap

Start every Dynatrace-backed task with a minimal capability and safety check:

```bash
# Install on macOS/Linux if needed
brew install dynatrace-oss/tap/dtctl

# Preferred auth
dtctl auth login --context dev --environment "https://<tenant>.apps.dynatrace.com"

# Verify config, context, connectivity, and auth
dtctl doctor

# Inspect or switch contexts
dtctl ctx
dtctl ctx current
dtctl ctx describe

# Let the agent discover available commands at runtime
dtctl commands --brief -o json
```

Use read-only contexts for production by default. Prefer project-local `.dtctl.yaml` without secrets when the repo needs shared Dynatrace automation.

## Repo discovery first

Do not guess the topology.

Before changing code or issuing queries, confirm:

1. How the demo runs.
   - Docker Compose: `docker-compose.yml`
   - Kubernetes: `kubernetes/opentelemetry-demo.yaml`
2. What the services are called.
   - Core services include `frontend`, `frontend-proxy`, `checkout`, `cart`, `product-catalog`, `payment`, `recommendation`, `shipping`, `currency`, `quote`, `email`, `fraud-detection`, `accounting`, `ad`, `flagd`, `otel-collector`, and `load-generator`.
3. What the critical path is.
   - Start at the user-facing entrypoint and trace downstream dependencies before adding work on the hot path.

## Default ODD workflow

### 1. Discover the runtime shape

- Identify the user-facing entrypoint and downstream call graph.
- Identify deployment identifiers you will need in Dynatrace: environment, namespace, deployment, service entity, workflow IDs, settings schema IDs.
- For Dynatrace artifacts, discover before editing:
  - `dtctl get workflows`
  - `dtctl get dashboards`
  - `dtctl get notebooks`
  - `dtctl get slos`
  - `dtctl get settings-schemas`

### 2. Establish a baseline before changing behavior

Use `dtctl` to validate and execute the smallest useful queries.

```bash
# Verify syntax before execution
dtctl verify query "fetch logs | limit 10"

# Run targeted queries
dtctl query "fetch logs | limit 10"
dtctl query "timeseries avg(dt.service.request.response_time), percentile(dt.service.request.response_time, 95), sum(dt.service.request.failure_count), from: now()-1h, interval: 1m, by:{dt.entity.service}"

# Wait for evidence during tests or rollout checks
dtctl wait query "fetch spans | filter trace.id == \"<trace-id>\"" --for=any --timeout 3m
```

Baseline expectations for hot-path work:

- request latency: p50, p95, p99
- error rate and failure count
- saturation: CPU, memory, restarts, throttling, queue depth where relevant
- dependency health and headroom

### 3. Apply a risk gate

Do not add synchronous work to a service that is already near saturation.

If baseline data shows limited headroom, prefer:

- narrower scope
- async execution
- caching
- feature flags
- workflow-based validation outside the hot path
- configuration changes in Dynatrace instead of code changes where possible

### 4. Implement with guardrails

- Add timeouts, bounded retries, and graceful degradation for new downstream calls.
- Reuse existing instrumentation conventions in the touched service.
- Avoid high-cardinality telemetry fields.
- Prefer stable identifiers for metrics, spans, logs, lookup tables, and settings objects.

### 5. Verify after the change

Repeat the same verified queries and compare deltas.

Use Dynatrace artifacts as part of the validation itself when useful:

- `dtctl apply -f dashboard.yaml` to publish a comparison dashboard
- `dtctl apply -f notebook.yaml` to persist analysis
- `dtctl create slo -f slo.yaml` or `dtctl apply -f slo.yaml` to formalize a rollback threshold
- `dtctl exec workflow <id> --wait` to run a post-deploy validation workflow

### 6. Operationalize the change

Elevate ODD beyond ad hoc querying.

Use `dtctl` to create or update the supporting runtime assets that make the change safe to own:

- workflows for automated post-deploy checks and notifications
- dashboards and notebooks for before/after analysis
- SLOs for explicit go/no-go criteria
- settings objects for OpenPipeline and other runtime configuration
- lookup tables for enrichment such as service ownership, error code mapping, or environment routing
- buckets for storage and retention changes when the change requires data-model support

## dtctl capability map for ODD

### Data plane

- `dtctl verify query ...` validates DQL before execution.
- `dtctl query ...` executes DQL with templates, files, live mode, and machine-readable output.
- `dtctl wait query ...` turns telemetry presence into an assertion for rollout checks and CI.
- `dtctl exec copilot nl2dql ...` helps generate candidate queries when the intent is clear but the DQL is not.
- `dtctl exec analyzer ...` supports forecast and anomaly analysis for capacity and rollout risk checks.

### Control plane

- `dtctl get|describe|edit|apply|delete workflow ...`
- `dtctl get|describe|edit|apply|share dashboard ...`
- `dtctl get|describe|edit|apply|share notebook ...`
- `dtctl get|create|apply|exec slo ...`
- `dtctl get settings-schemas`, `dtctl get|create|update|apply settings ...`
- `dtctl get|create|delete lookup ...`
- `dtctl get|create|delete bucket ...`

### Collaboration and automation plane

- `dtctl exec workflow ... --wait` for runbooks and automated validation
- `dtctl logs workflow-execution ... --follow` for workflow debugging
- `dtctl share dashboard ...` and `dtctl share notebook ...` for team visibility
- `dtctl open intent ...` for deep links into specific Dynatrace app views

## Risk tiers

- Tier 0: docs, refactors, or no behavior change -> minimal validation
- Tier 1: non-hot-path behavior change -> targeted tests plus basic telemetry verification
- Tier 2: hot-path change, sync downstream call, retry change, payload change, or config that can alter runtime behavior -> baseline comparison required, explicit rollback triggers required, and Dynatrace validation expected

## Production readiness in design work

When a design adds or changes service-to-service calls, retry behavior, timeouts, circuit breakers, ingestion configuration, routing, or alerting semantics, perform a production readiness assessment before asking for approval.

Minimum assessment:

1. Identify affected services and Dynatrace artifacts.
2. Capture baseline latency, error, and saturation signals.
3. Compare the design assumptions to actual production or staging behavior.
4. Decide go, no-go, or go-with-guardrails.
5. Define rollback thresholds.

Where useful, make the assessment durable:

- store the queries in `.dql` files and verify them with `dtctl verify query -f ... --fail-on-warn`
- export notebooks or dashboards as YAML and manage them with `dtctl apply`
- encode thresholds in SLOs or workflows instead of leaving them as prose only

## Query hygiene

- Verify before execute.
- Keep queries narrow first, then widen only if needed.
- Use short windows for triage and longer windows for baseline.
- Do not mix entity metadata queries with metric timeseries expectations.
- Use aliases in `summarize` and `sort`.
- If results are empty, state `0 records` and propose the next smallest checks.

Minimal examples:

```bash
dtctl verify query -f - <<'EOF'
fetch logs
| filter timestamp > now() - 1h
| summarize logCount = count(), by:{k8s.namespace.name}
| sort logCount desc
EOF

dtctl query -f - <<'EOF'
fetch spans
| filter timestamp > now() - 30m
| summarize p95 = percentile(duration, 95), count = count(), by:{dt.entity.service, span.name}
| sort p95 desc
EOF
```

## Safe mutation rules

- Default to read-only contexts for production.
- Use `dtctl apply --dry-run`, `dtctl diff`, and `--show-diff` before mutating shared Dynatrace resources.
- Prefer templated manifests over imperative one-off changes for anything that should be repeatable.
- Never commit real tenant URLs, tokens, user identifiers, or customer data.
- Do not create production workflows, settings objects, or lookups without explicit scope and rollback guidance.

## Local fallback workflow

If Dynatrace access is unavailable:

- run the demo via Docker Compose or Kubernetes
- generate traffic with `load-generator`
- compare before and after p95 latency and error rate for the affected entrypoint
- run the narrowest relevant service tests
- still write the `dtctl` commands or manifests you would use once Dynatrace access is restored

## Definition of done

For non-trivial changes, the final outcome should include:

- correctness validation for success and failure paths
- explicit timeouts and failure handling for new dependencies
- telemetry that can prove the change worked
- a baseline and after-change comparison plan
- rollback triggers grounded in actual signals
- when appropriate, persistent Dynatrace assets created or updated through `dtctl`