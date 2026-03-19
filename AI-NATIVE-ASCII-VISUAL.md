# AI-Native Lifecycle — ASCII Visual

This is the current end-to-end visual for the three-agent operating model:

- Kiro = design + all GitOps changes
- AWS DevOps Agent = pipeline/deploy context and execution
- Observability Operator = Evidence, Gate, Triage
- GitHub Issues = coordination and lineage
- Dynatrace = runtime trigger

```text
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                                AI-NATIVE CLOSED LOOP                                        │
│                   (AWS + Kiro + Dynatrace + GitHub Issues)                                  │
└──────────────────────────────────────────────────────────────────────────────────────────────┘

PHASE 1: DESIGN TIME                                                                     Trigger: Kiro
────────────────────────────────────────────────────────────────────────────────────────────────────────

  Human Request
  (GitHub Issue #FR)
         │
         ▼
  ┌───────────────┐             calls (Evidence mode)              ┌──────────────────────────────┐
  │     KIRO      │───────────────────────────────────────────────►│   OBSERVABILITY OPERATOR     │
  │ spec/design   │◄───────────────────────────────────────────────│        EVIDENCE MODE         │
  │ + gitops plan │    baseline, headroom, risk, readiness verdict │ dtctl + Dynatrace evidence   │
  └───────────────┘                                                └──────────────────────────────┘
         │
         ▼
  Design Gate
  (go / go-with-guardrails / no-go)


PHASE 2: DELIVERY TIME                                                                 Trigger: Pipeline
────────────────────────────────────────────────────────────────────────────────────────────────────────

  ┌───────────────┐        PR with app+infra changes         ┌──────────────────────────────┐
  │     KIRO      │──────────────────────────────────────────►│       CI/CD PIPELINE         │
  │ implements    │                                           │   (AWS DevOps Agent lane)    │
  │ behind flags  │◄──────────────────────────────────────────│ builds, tests, deploy steps  │
  └───────────────┘          status + artifacts              └──────────────────────────────┘
                                                                │
                                                                │ calls (Gate mode)
                                                                ▼
                                                       ┌──────────────────────────────┐
                                                       │   OBSERVABILITY OPERATOR     │
                                                       │          GATE MODE           │
                                                       │ telemetry checks + verdict   │
                                                       └──────────────────────────────┘
                                                                │
                                                                ▼
                                                     Delivery Gate (pass / warn / fail)


PHASE 3: RUNTIME                                                                        Trigger: Dynatrace
────────────────────────────────────────────────────────────────────────────────────────────────────────

                              live telemetry / anomalies
                ┌─────────────────────────────────────────────────────────┐
                │                        DYNATRACE                        │
                │          detects incidents and invokes triage           │
                └─────────────────────────────────────────────────────────┘
                                         │
                                         ▼
                              ┌──────────────────────────────┐
                              │   OBSERVABILITY OPERATOR     │
                              │         TRIAGE MODE          │
                              │ correlate: obs + AWS context │
                              │ classify lane A / B / C      │
                              └──────────────────────────────┘
                                         │
                                         ▼
                       Creates structured remediation GitHub Issue (#INC)
                                         │
                                         ▼
                                    ┌───────────┐
                                    │   KIRO    │
                                    │ picks up  │
                                    │ and acts  │
                                    └───────────┘


FAILURE / REMEDIATION BRANCHES
────────────────────────────────────────────────────────────────────────────────────────────────────────

  Lane A: Infrastructure
  ──────────────────────
  Signal: saturation, scaling limits, config drift
  Flow:   #INC(lane-a) -> Kiro creates infra-as-code PR -> Pipeline -> Gate -> Runtime verify

  Lane B: Code Regression
  ───────────────────────
  Signal: new exceptions, latency regression, retry storms
  Flow:   #INC(lane-b) -> Kiro creates code fix PR -> Pipeline -> Gate -> Runtime verify

  Lane C: Design Mismatch
  ───────────────────────
  Signal: implementation matches plan, but assumptions fail in production
  Flow:   #INC(lane-c) -> Kiro updates design/spec -> Evidence mode again -> new implementation cycle


COORDINATION BACKBONE (GitHub Issues)
────────────────────────────────────────────────────────────────────────────────────────────────────────

  #FR (feature request)
    └─ Design outputs, approvals, rollout contract
       └─ #INC (runtime incident, child/related)
          ├─ labels: incident, lane-a|lane-b|lane-c, service, severity
          ├─ evidence: baseline vs detected deltas
          ├─ recommended action: concrete Kiro task
          └─ verification: DQL checks / dashboard links


SINGLE-LINE SUMMARY
────────────────────────────────────────────────────────────────────────────────────────────────────────

  Kiro triggers evidence at design time, pipeline triggers delivery gate checks,
  Dynatrace triggers runtime triage; GitHub issues carry lineage, and Kiro executes all changes via GitOps.
```
