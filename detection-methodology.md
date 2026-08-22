# Detection Methodology

Hybrid detection: no single technique is trusted alone.

## Layers

| Layer | Mechanism | Example | Strength | Weakness |
|---|---|---|---|---|
| Rules | Deterministic pattern match (YAML rule definitions, evaluated per event) | Shell spawned from a web-server process; connection to a known-bad IP | Fast, explainable, zero false-positive-by-design for well-written rules | Only catches known patterns |
| Statistics | Per-asset baseline (mean/std, rolling window) | Sudden spike in outbound connections | Catches unknown deviations from *this* asset's normal | Needs baseline warm-up; noisy on volatile assets |
| ML (Isolation Forest) | Multivariate anomaly scoring across engineered features | Combination of rare executable + rare destination + odd timing | Catches subtle multi-feature anomalies rules/stats miss individually | Needs feature engineering, less directly explainable (mitigated via top-feature contributions) |
| Behavior Correlation | Groups findings across time/asset/actor into a narrative | login failure → new process → outbound connection within 5 min | Turns isolated low-confidence findings into a high-confidence chain | Requires correct time-windowing, tunable |
| Threat Intelligence | Indicator match (IP/domain/hash) against `threat_indicators` | Connection to indicator with `confidence=high` | High precision when matched | Coverage limited to seeded/available intel for MVP |
| Asset Criticality | Weighting multiplier | Same finding on a database server vs. a test VM | Focuses attention where impact is highest | Requires accurate asset classification by the org |

## How Signals Combine

Each layer emits a **finding**, not a verdict:

```json
{
  "finding_id": "uuid",
  "source": "rule | statistical | ml | correlation | threat_intel",
  "weight": 0.0,
  "confidence": 0.0,
  "description": "...",
  "evidence_event_ids": ["..."]
}
```

Findings for the same asset within a correlation window are grouped by the
Correlation Service into a behavior cluster before being handed to the Risk
Engine (see `docs/response/response-model.md` for what happens next).
Rules can fire directly into a high-confidence finding without requiring ML
agreement (e.g., a known-bad-hash file write); ML/statistical findings
alone require corroboration or a high anomaly score before contributing
meaningfully to risk (avoiding "AI said so" false positives).

## Rule Format (MVP)

```yaml
id: rule-priv-esc-shell-from-webproc
description: Shell spawned by a public-facing web server process
match:
  event_type: process_start
  parent_executable_in: ["nginx", "apache2", "gunicorn"]
  executable_in: ["/bin/sh", "/bin/bash", "/bin/dash"]
severity: high
mitre_technique: T1059
```

Rules are versioned, tenant-overridable, and stored in the database (not
hardcoded in application code) so `SECURITY_ENGINEER` users can tune them
without a deployment.
