# AI / ML Architecture

AI is an **assisting signal**, not the sole authority for high-impact
decisions. Every AI output feeds the deterministic Risk Engine as one
weighted, explainable component alongside rules, behavior correlation, and
threat intel — it never unilaterally creates or closes an incident.

## Pipeline

```text
Telemetry → Feature Extraction → Per-Asset Rolling Baseline →
Isolation Forest Anomaly Scoring → Behavior Analysis / Correlation →
Confidence Estimation → Risk Signal (score + top contributing features)
```

## Features (MVP set, per asset, rolling window)

- Process: rate of new process starts, rarity of executable path (seen
  before on this asset?), parent/child anomalies (e.g., office app spawning
  a shell), average process lifetime.
- Network: rate of new outbound destinations, connections to non-standard
  ports, DNS query entropy/rarity, volume deltas.
- Auth: failed-login rate, login time-of-day deviation, new source IP for
  a known user.
- File: rate of file modifications, mass-rename/mass-modify bursts,
  modifications to sensitive paths.

## Baseline

Rolling 7-day per-asset statistical baseline (mean/std per feature),
recomputed daily. Cold-start assets (new agent) use organization-wide
baseline for the first 48h and are flagged as "baseline warming" so the
UI doesn't present low-confidence early scores as equally reliable.

## Model

**Isolation Forest** (scikit-learn), one model per organization (or per
asset class if data volume justifies it later), retrained on a schedule
(e.g., nightly) from the last N days of feature vectors, versioned
(`model_name`, `model_version` stored with every prediction for
reproducibility).

Why Isolation Forest for MVP: no labeled attack data available (a real
constraint), unsupervised, computationally cheap, and its per-feature
path-length contribution can be surfaced as a rough explainability signal.
Evaluated and rejected for MVP: deep autoencoders (needs more data and
GPU infra than justified), supervised classifiers (no reliable labels),
clustering-only approaches (weaker for point-anomaly detection at this
scale).

## Explainability

Every AI prediction is stored with:

```json
{
  "anomaly_score": 0.82,
  "confidence": 0.71,
  "top_features": [
    {"feature": "new_executable_path", "contribution": 0.34},
    {"feature": "outbound_connection_rarity", "contribution": 0.27}
  ],
  "model_version": "if-2026-08-01"
}
```

The UI never shows a bare "AI detected malware." It always shows the
observed signals, why they're unusual relative to baseline, which features
contributed, and the resulting confidence — per Section 7 of the master
instructions.

## False-Positive Handling

- Confidence below a configurable threshold routes the finding to
  "monitor only," not incident creation.
- Analyst feedback (mark incident as false positive) is stored and
  factored into the next retraining cycle's baseline exclusion list —
  not a live online-learning loop for MVP (deliberately, to keep behavior
  auditable/reproducible).

## LLM Use (bounded)

An LLM (external API or local) may be used strictly for: incident
summarization, natural-language explanation of a risk score's components,
and draft report text. It never independently decides risk, severity, or
triggers a response action — those remain deterministic engine outputs.
