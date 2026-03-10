# AI-Native Lifecycle

This document defines an AI-native software lifecycle for any application, not just this repository.

It is designed for environments where AI agents assist heavily and may operate with supervised or near-autonomous execution, while humans retain policy authority and explicit go or no-go control at the highest-risk transitions.

## Core idea

An AI-native lifecycle has two native coordination channels:

- the system of record: specifications, code, policies, release contracts, and decisions
- the observability platform: runtime truth, change impact evidence, anomaly detection, and feedback into the next work cycle

The repo is the governance backbone. Observability is the nervous system.

The lifecycle is not linear. It is a closed loop:

1. design and govern
2. build and deliver
3. operate and learn
4. feed new signals back into design

## Operating principles

- Observability is a design-time concern, not a post-implementation task.
- Governance, compliance, and production readiness are evaluated before code generation begins.
- Delivery is gated by evidence, not intent.
- Runtime incidents are routed to the right lane: infrastructure, code, or design.
- Feature flags, canaries, and rollback criteria are part of the change definition.
- Agents recommend and execute within scope; humans approve risk-bearing transitions by exception.

## Lifecycle phases

### 1. Design time

Goal: turn a user request or signal into a production-aware, policy-compliant implementation plan.

Required agent responsibilities:

- enrich the spec with runtime evidence from the observability platform
- identify impacted components, dependencies, and user journeys
- evaluate governance, security, compliance, and data handling constraints
- assess current load, saturation, error budget status, and likely change impact
- define observability, rollout, and rollback before implementation starts

Required artifacts:

- mission brief or product spec
- technical design
- observability design
- compliance and governance assessment
- production readiness assessment

Design-time gate must answer:

- What services, functions, workflows, or data pipelines are affected?
- What is the current production or staging baseline?
- Is any affected component already close to saturation or error budget exhaustion?
- What telemetry will prove this change is healthy?
- What compliance or governance controls apply?
- What rollout mechanism will contain blast radius?
- What thresholds will trigger automatic halt or rollback?

Mandatory design-time checks:

- Governance: approvals, ownership, change scope, auditability, policy fit
- Observability: traces, metrics, logs, dashboards, SLOs, alerts, runtime verification plan
- Compliance: security, privacy, data residency, regulated workflow impact, evidence capture
- Production readiness: dependency load, latency budgets, resource headroom, failure mode analysis, rollback path

Design-time verdicts:

- PASS: safe to move to delivery
- PASS WITH GUARDRAILS: allowed only with reduced scope, flagging, or rollout constraints
- FAIL: return to design with explicit blockers

### 2. Delivery time

Goal: generate, validate, and ship the change without introducing regressions.

Required agent responsibilities:

- implement the change behind a feature flag when feasible
- add instrumentation required by the observability design
- run correctness, security, and regression checks
- validate resource footprint, performance, and error behavior in staging or pre-production
- prepare a release contract with rollout and rollback conditions

Required artifacts:

- implementation PR or change set
- test and evaluation results
- release contract
- rollout plan
- canary or flag configuration
- updated dashboards, notebooks, workflows, or SLOs as needed

Delivery-time gate must answer:

- Does the change meet functional acceptance criteria?
- Did resource usage regress beyond budget?
- Are new error patterns present in logs, traces, or metrics?
- Is performance within allowed delta?
- Is telemetry flowing and usable within minutes of deployment?
- Is the change protected by feature flags, canarying, or another blast-radius control?

Mandatory delivery-time checks:

- Correctness: tests, contract checks, failure-path validation
- Regression: CPU, memory, latency, throughput, error logs, trace errors
- Observability flow: traces and metrics visible in staging, structured logs correlated by trace
- Release safety: feature flag or canary strategy, halt and rollback rules, ownership assigned

Delivery-time verdicts:

- PASS: safe for progressive rollout
- PASS WITH NOTES: ship allowed, but monitor tighter or cap rollout
- FAIL: return to implementation
- FAIL AND RE-DESIGN: assumptions invalid, return to design

### 3. Runtime

Goal: monitor the change in production, detect deviations fast, and route remediation to the right execution lane.

Required agent responsibilities:

- monitor release health, SLOs, saturation, error patterns, and user-impact signals
- compare live behavior to baseline and rollout expectations
- triage incidents and anomalies
- initiate mitigation through the correct owner agent
- file a structured signal back into the next design cycle when the issue is architectural or product-level

Runtime gate must answer:

- Is the feature healthy enough to continue rollout?
- Is the incident primarily operational, implementation, or design-related?
- Can the issue be mitigated safely without code changes?
- Do we need to freeze, roll back, or re-enter the design loop?

Mandatory runtime checks:

- release health during canary and post-GA
- burn rate and error budget consumption
- infra saturation and cost anomalies
- new error classes or retry storms
- feature adoption and business outcome signals

## Incident routing model

Every runtime issue must be classified into one of three lanes.

### Lane A: DevOps or infrastructure

Route to a DevOps or infrastructure agent when signals point to:

- capacity saturation
- autoscaling or scheduling issues
- network, ingress, service mesh, or DNS problems
- storage or database resource exhaustion
- deployment configuration drift
- environment policy or secret/config distribution issues

Expected outputs:

- infrastructure change proposal or direct remediation within approved scope
- updated runtime configuration
- validation against release health after mitigation

### Lane B: Code or implementation

Route to a coding or application remediation agent when signals point to:

- introduced exceptions or new error classes
- latency regressions tied to the new logic
- memory leaks, connection leaks, retry bugs, or hot loops
- missing or broken instrumentation
- feature-flag behavior defects

Expected outputs:

- remediation spec
- code fix behind existing rollout controls
- repeat of delivery-time validation

### Lane C: Design or architecture

Route back to design when signals show the implementation matches the plan but the plan was wrong.

Examples:

- dependency cannot sustain the new call pattern
- latency or throughput assumptions were invalid
- compliance or governance assumptions were incomplete
- the rollout strategy is not safe enough for the actual blast radius
- the business outcome creates unplanned load or usage patterns

Expected outputs:

- updated design
- revised observability design and production readiness assessment
- new mission or change contract

## Agent roles in the lifecycle

These roles can be implemented as separate agents or combined into a smaller fleet. The separation matters more than the runtime.

### Design-time agents

- Spec Enrichment Agent
- Architecture and Production Readiness Agent
- Observability Design Agent
- Governance and Compliance Agent

### Delivery-time agents

- Coding Agent
- Observability Compliance Agent
- Delivery Evaluator
- Release and Rollout Agent

### Runtime agents

- Release Health Agent
- Incident Triage Agent
- DevOps or Infrastructure Agent
- Remediation Spec Agent

## Evidence model

Every major agent decision must be supported by evidence.

Accepted evidence sources:

- production or staging telemetry
- verified tests and benchmark runs
- static policy checks
- deployment and rollout state
- cost and capacity data

Every decision should produce:

- a verdict
- the evidence used
- the threshold evaluated
- the next action

## Observability contract for the lifecycle

The observability platform is not just a monitor. It is an execution input and a governance feedback channel.

Agents must consume observability before deciding:

- production baselines
- current headroom
- recent anomalies
- dependency health

Agents must emit observability after acting:

- spans for decisions, tool calls, and workflow steps
- metrics for throughput, latency, errors, rollout progress, and agent outcomes
- logs with structured context and correlation IDs

## Human oversight model

Humans do not need to approve every step, but they should own the highest-risk transitions:

- design approval for high-impact changes
- release approval for production rollout beyond canary
- governance exceptions
- major incident escalation or rollback override

## Minimum reusable templates

Any application adopting this lifecycle should have templates for:

- spec or mission brief
- technical design with observability design section
- production readiness assessment
- release contract
- incident triage report
- remediation spec
- post-incident retrospective

## Definition of done

A change is only done when:

- it passed design-time governance, observability, compliance, and production-readiness checks
- it passed delivery-time regression and rollout safety gates
- it was monitored in runtime with explicit success and rollback criteria
- any incident or anomaly can be routed to infrastructure, code, or design without ambiguity
- the lifecycle produced reusable evidence for the next cycle