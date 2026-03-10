# AI-Native Lifecycle — Steering

Use this steering for any task that describes an end-to-end lifecycle with AI agents, not just a code change.

## Intent

Operate as if software delivery is a closed loop between:

- design and governance
- delivery and release
- runtime monitoring and remediation

Observability is not just monitoring. It is a decision input before design, a quality gate during delivery, and a feedback channel during runtime.

## Required lifecycle behavior

### At design time

Before implementation begins, agents must ensure:

- governance and ownership are explicit
- observability design exists
- compliance and policy constraints are evaluated
- production readiness is assessed using current runtime data when available
- current load, headroom, and potential change impact are checked

If these are incomplete, do not proceed to coding.

### At delivery time

Before rollout, agents must ensure:

- correctness validation passes
- regression risk is evaluated for resource footprint, error logs, and performance
- instrumentation is present and telemetry is flowing
- rollout is bounded by feature flags, canarying, or equivalent controls
- rollback thresholds are defined

### At runtime

After deployment, agents must:

- monitor release health against baseline and thresholds
- detect incidents or degradations
- triage issues into infrastructure, code, or design lanes
- route infrastructure issues to DevOps or SRE agents
- route implementation regressions back to coding agents
- route broken assumptions back into design and readiness analysis

## Default escalation model

- Infrastructure issue -> DevOps or SRE remediation
- Code issue -> remediation spec plus code fix
- Design issue -> re-enter design with updated production evidence

## Demo expectation

When building or narrating a demo, show the entire loop:

1. User creates spec
2. Agent enriches with runtime data
3. Design is gated by governance, observability, compliance, and production readiness
4. Agent implements behind release controls
5. Delivery gates validate correctness and regressions
6. Progressive rollout begins
7. Runtime monitoring confirms health or triggers incident routing
8. The outcome flows either to promotion, operational mitigation, or re-entry into design and coding

## Primary references

- `AI-NATIVE-LIFECYCLE.md`
- `AI-NATIVE-DEMO-FLOW.md`
- `AGENTS.md`
- `.kiro/steering/dynatrace-odd.md`