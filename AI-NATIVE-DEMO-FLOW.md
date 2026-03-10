# AI-Native Demo Flow

This document defines a reusable demo flow for showing a full AI-native lifecycle on any application.

It is designed to demonstrate autonomous or near-autonomous agents working end to end, with humans intervening mainly at governance and go or no-go boundaries.

## Demo objective

Show a full closed loop:

1. a request enters as a spec
2. agents enrich it with runtime evidence
3. the change is designed with governance and production awareness
4. code is generated and validated behind rollout controls
5. the change is deployed progressively
6. runtime health is monitored
7. an issue is either mitigated operationally or routed back to code or design

## Recommended demo scenario

Use a change that is easy to explain and naturally exercises risk gates.

Best default:

- Add a new synchronous dependency call or a behavior change on a user-facing path, protected by a feature flag.

Why this works:

- it creates a real design-time production-readiness question
- it forces observability design before coding
- it enables a canary or feature-flag rollout
- it creates a plausible runtime failure branch

Examples for any app:

- ecommerce: add recommendation lookup during checkout
- banking: add fraud scoring before payment confirmation
- SaaS: add entitlement validation before report export
- support app: add summarization step before case submission

## Demo actors

### Human roles

- product owner or requestor
- approving engineer, architect, or SRE lead

### Agent roles

- Spec Enrichment Agent
- Production Readiness Agent
- Governance and Compliance Agent
- Coding Agent
- Delivery Evaluator
- Release and Rollout Agent
- Release Health Agent
- Incident Triage Agent
- DevOps Agent
- Remediation Spec Agent

## Demo phases

### Phase 1. Request and enrichment

Story:

- A human creates a short spec.
- The Spec Enrichment Agent identifies impacted components and enriches the request with runtime data.

Show on screen:

- original spec
- dependency map or impacted service list
- baseline latency, error, and saturation evidence

Evidence to show:

- baseline query or dashboard
- current error budget or rollout risk summary

Key message:

- The spec is not accepted as-is. It becomes runtime-aware before design begins.

### Phase 2. Design-time governance gate

Story:

- The design agent creates a technical design.
- Governance and Compliance Agents evaluate policy, compliance, and production readiness.

Show on screen:

- technical design sections
- observability design
- production readiness assessment
- gate verdict

Required visible checks:

- governance and ownership
- observability plan
- compliance and data handling impact
- current load and change impact
- rollout and rollback strategy

Key message:

- Design is approved only if it is observable, compliant, and safe enough to test in production.

### Phase 3. Build and validation

Story:

- The Coding Agent implements the change behind a feature flag.
- The Delivery Evaluator validates correctness and regression risk.

Show on screen:

- code change summary
- tests and evaluation verdicts
- instrumentation added
- feature flag or canary config

Required visible checks:

- correctness tests
- error-log regression check
- performance comparison
- resource footprint comparison
- telemetry flow confirmation

Key message:

- The build is not considered shippable until the observability and regression gates pass.

### Phase 4. Progressive rollout

Story:

- The Release Agent deploys to staging and then to production as a canary or flagged rollout.
- The Release Health Agent watches the change in real time.

Show on screen:

- rollout contract
- canary percentage or flag state
- live health dashboard or notebook
- go or no-go recommendation

Required visible checks:

- live error rate
- live p95 latency
- resource saturation
- feature-level business metric if available

Key message:

- Shipping is a monitored experiment, not a blind push.

### Phase 5. Runtime branch A: healthy rollout

Story:

- Signals stay within thresholds.
- The Release Health Agent recommends promotion to broader rollout or GA.

Show on screen:

- before and after comparison
- threshold summary
- promotion decision

Key message:

- AI agents can handle the full delivery loop with evidence and bounded autonomy.

### Phase 6. Runtime branch B: issue detected

Story:

- The Release Health Agent detects a problem.
- The Incident Triage Agent classifies it and routes it to the right execution lane.

Show on screen:

- anomaly or incident signal
- triage classification
- handoff target

Possible branches:

- infrastructure branch: DevOps Agent changes infra or config
- code branch: Remediation Spec Agent sends work back to coding
- design branch: architecture assumptions invalid, return to design

Key message:

- Runtime is part of the lifecycle, not the end of it.

## Suggested issue branches for the demo

### Branch 1: infrastructure issue

Use when you want to show DevOps autonomy.

Example:

- canary increases CPU and queue latency because autoscaling thresholds are too conservative

Flow:

1. Release Health Agent detects saturation
2. Incident Triage Agent classifies as infrastructure
3. DevOps Agent updates scaling or resource limits
4. Release Health Agent confirms stabilization
5. rollout resumes

### Branch 2: code regression

Use when you want to show a return to implementation.

Example:

- new dependency retry logic causes error storms and elevated latency

Flow:

1. Release Health Agent detects latency and error regression
2. Incident Triage Agent classifies as code issue
3. Remediation Spec Agent writes the fix brief
4. Coding Agent patches code behind the same flag
5. Delivery and rollout gates run again

### Branch 3: design flaw

Use when you want to show the strongest form of AI-native learning.

Example:

- the dependency cannot sustain the synchronous call pattern at real traffic levels

Flow:

1. runtime evidence contradicts design assumptions
2. Incident Triage Agent classifies as design issue
3. updated production-readiness assessment is created
4. solution changes to async, cached, or degraded mode
5. lifecycle restarts from design

## Demo assets to prepare

For the demo to feel complete, prepare these artifacts ahead of time:

- a short initial spec
- a technical design template with observability and production readiness sections
- a release contract template
- one dashboard or notebook showing baseline and live release health
- a workflow or automation that runs rollout checks
- a signal or incident template for runtime feedback

## Minimal success criteria for the demo

The demo is successful if the audience sees:

- design-time use of runtime evidence
- explicit governance, compliance, observability, and production-readiness gating
- delivery-time regression gating and rollout safety
- runtime monitoring tied to clear thresholds
- automated triage to DevOps, coding, or design
- the loop closing back into the next mission or remediation cycle

## Narration guide

Use these lines to keep the story crisp:

- We do not let agents design blind. They start from runtime truth.
- Observability is part of the design contract, not a post-deploy task.
- We do not ship code; we ship bounded experiments with rollback criteria.
- Runtime issues do not become generic tickets. They are classified and routed to the correct agent lane.
- The loop closes when runtime evidence becomes the next design input.