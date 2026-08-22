# Incident & Response Engine

## Risk Engine

Deterministic, weighted, and always explainable — even though some inputs
come from ML.

```text
risk_score =
    w1 * anomaly_score        (ML)
  + w2 * behavior_severity     (correlation cluster severity)
  + w3 * asset_criticality     (1-5 normalized)
  + w4 * threat_intel_match    (0 or confidence-weighted)
  + w5 * historical_behavior   (repeat offender factor)
  * confidence_multiplier      (reduces score when signal confidence is low)
```

Weights and thresholds live in `security_policies.risk_thresholds` (per
tenant, editable by `SECURITY_ENGINEER+`), not hardcoded — the formula
itself is documented here and in the dashboard so a score of "94" is
always traceable to its components (see `ai-ml-design.md` explainability
example).

| Range | Severity |
|---|---|
| 0–20 | Normal |
| 21–40 | Low |
| 41–60 | Medium |
| 61–80 | High |
| 81–100 | Critical |

## Incident State Machine

```text
Detected → Correlated → Classified → RiskAssessed → Created →
Investigating → Contained → Remediating → Verifying → Recovered →
Reported → Closed
```

(Investigating → Closed directly is allowed for false positives, with a
required resolution reason logged.) Every transition writes an
`incident_events` row with actor, timestamp, and reason — the state
machine itself is enforced in the Incident Engine service, not left to
ad-hoc status field updates from the API.

Incidents are deduplicated by a correlation key (asset + behavior cluster
signature within a time window) so a burst of related findings creates one
incident with a growing timeline, not dozens of duplicates.

## MITRE ATT&CK Mapping

Each rule and behavior-cluster template carries one or more MITRE
technique IDs (e.g., `T1059` — Command and Scripting Interpreter). The
Incident Engine aggregates all technique IDs from contributing findings
onto the incident record for MITRE-mapped visualization on the dashboard.

## Response Engine

```text
Detection → Risk → Policy Evaluation → Decision →
[Approval if required] → Action Execution → Verification → Recovery
```

Policy model (per tenant, configurable):

| Risk | Default Action |
|---|---|
| Low | Monitor only |
| Medium | Alert analyst |
| High | Recommend containment, require human approval |
| Critical | Controlled autonomous containment **only if** policy explicitly enables it for that action type; otherwise require approval |

Supported actions (MVP): `ISOLATE_ENDPOINT`, `BLOCK_IP`, `TERMINATE_PROCESS`,
`QUARANTINE_FILE`, `COLLECT_FORENSICS`. Domain blocking, session
disable, and full remediation/restore workflows are architected but may be
simulated against the lab environment rather than a real network device for
MVP demo purposes — this is documented, not hidden (`MVP_SCOPE.md`).

Every `response_actions` row requires: `actor`, `reason`, `target_asset_id`,
`requires_approval`, `approved_by` (nullable until approved),
`executed_at`, `status`, and a rollback/recovery reference where
applicable (e.g., isolation records the pre-isolation network zone so
recovery can restore it). Nothing executes without passing through this
audited record — there is no "fire and forget" action path.

## Containment Model

Isolation is **localized**: the target asset's zone membership moves to
Quarantine; no other asset or zone is affected (see `ARCHITECTURE.md` §9).
Recovery reverses this once verification confirms the asset is clean.
