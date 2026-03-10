# AI Agent Guide for OpenTelemetry Demo

This repository expects observability-driven development. For any non-trivial behavior change, do not stop at code correctness. Prove the change against runtime signals and make the validation repeatable.

For full lifecycle guidance beyond this repository's software topology, also use:

- `AI-NATIVE-LIFECYCLE.md`
- `AI-NATIVE-DEMO-FLOW.md`

## Core rule

Prefer `dtctl` as the default Dynatrace interface.

Use it for:

- query verification and execution
- runtime discovery across environments
- workflows, dashboards, notebooks, and SLOs
- settings and OpenPipeline configuration
- lookup tables, buckets, and other shared operational assets

Dynatrace MCP is optional. If both are available, default to `dtctl` unless MCP is clearly faster for a narrow read-only task.

If Dynatrace or `dtctl` is unavailable, say so explicitly and fall back to a local baseline.

## AI-native lifecycle

Use a full lifecycle model, not a code-only model.

### Design time

Before implementing, agents should ensure:

- governance and ownership are clear
- observability is designed before coding
- compliance and policy impact are evaluated
- production readiness is checked against current runtime evidence when available
- current load and likely change impact are understood

### Delivery time

Before rollout, agents should ensure:

- correctness and regression checks pass
- resource footprint, performance, and error behavior are evaluated
- telemetry is present and visible
- rollout is bounded by feature flags, canaries, or equivalent controls
- rollback thresholds are explicit

### Runtime

After deployment, agents should:

- monitor release health against baseline and thresholds
- classify incidents into infrastructure, code, or design lanes
- route infra issues to DevOps or SRE work
- route code regressions back to remediation and implementation
- route broken assumptions back into design and production-readiness review

Observability is both an input to decisions and an output of actions.

## Repo topology

Discover the actual runtime shape before changing code.

- Local deployment paths: `docker-compose.yml` and `kubernetes/opentelemetry-demo.yaml`
- User-facing entrypoints usually start at `frontend` and `frontend-proxy`
- Core services include `checkout`, `cart`, `product-catalog`, `payment`, `recommendation`, `shipping`, `currency`, `quote`, `email`, `fraud-detection`, `accounting`, `ad`, `flagd`, `otel-collector`, and `load-generator`
- Use service READMEs when you need build, test, or instrumentation details for one service

Do not guess service names, deployment names, namespaces, or entity IDs.

## dtctl bootstrap

Before using Dynatrace data or mutating Dynatrace resources:

```bash
dtctl doctor
dtctl ctx
dtctl commands --brief -o json
```

Typical setup flow:

```bash
dtctl auth login --context dev --environment "https://<tenant>.apps.dynatrace.com"
dtctl doctor
dtctl ctx describe
```

Prefer read-only contexts for production. Prefer project-local `.dtctl.yaml` without secrets if the repo needs shared automation.

## ODD workflow

1. Discover the critical path.
2. Capture a baseline before changing behavior.
3. Reject unsafe designs early if the target service has little headroom.
4. Implement with timeouts, retries, graceful degradation, and telemetry.
5. Re-run the same checks after the change.
6. Make useful validation artifacts durable in Dynatrace.

## What durable validation means

Use `dtctl` to create or update the things that let the team own the change after merge:

- workflows for post-deploy checks and notifications
- dashboards for service health and rollout comparison
- notebooks for investigation and before/after analysis
- SLOs for explicit rollback thresholds
- settings objects for OpenPipeline and other platform configuration
- lookup tables for ownership, routing, or enrichment data

## Query workflow

Always verify DQL before executing it.

```bash
dtctl verify query "fetch logs | limit 10"
dtctl query "fetch logs | limit 10"
```

For rollout assertions and automated checks:

```bash
dtctl wait query "fetch spans | filter trace.id == \"<trace-id>\"" --for=any --timeout 3m
```

For agent bootstrap and command discovery:

```bash
dtctl commands --brief -o json
```

Keep DQL narrow first. Use short time windows for triage, then widen carefully for baselines.

## Mutating Dynatrace resources

Treat workflows, dashboards, notebooks, settings, lookups, and buckets as shared infrastructure.

Rules:

- prefer `apply` over one-off imperative mutations when the resource should be reproducible
- use `--dry-run`, `diff`, and `--show-diff` before mutating shared resources
- promote through safer contexts before production
- never commit tenant secrets, tokens, or customer identifiers
- document rollback triggers whenever you change runtime config or automation

Useful patterns:

```bash
dtctl apply -f workflow.yaml --dry-run
dtctl diff -f workflow.yaml
dtctl apply -f dashboard.yaml --show-diff
dtctl exec workflow <id> --wait
dtctl logs workflow-execution <id> --follow
```

## Local fallback

If Dynatrace is not available:

- run the demo through Docker Compose or Kubernetes
- use `load-generator` to create consistent traffic
- compare before and after p95 latency and error rate for the affected entrypoint
- run the narrowest relevant service tests
- still specify the `dtctl` commands or manifests that should be used once access exists

## Definition of done

For non-trivial changes, the final result should include:

- correctness validation for happy path and failure modes
- telemetry or telemetry changes needed to evaluate impact
- a before/after verification plan
- rollback triggers grounded in actual signals
- when relevant, durable Dynatrace assets created or updated with `dtctl`