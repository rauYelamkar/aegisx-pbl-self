# ADR-0006: Isolation Forest as the initial anomaly detection model

## Status
Accepted

## Decision
Use scikit-learn's Isolation Forest, trained per-organization on rolling
feature windows, as the MVP unsupervised anomaly detector (see
`docs/ai/ai-ml-design.md` for the full feature/pipeline design).

## Alternatives Considered
- **Autoencoder-based anomaly detection:** can capture more complex
  nonlinear structure but requires more data and (ideally) GPU infra than
  is justified for an MVP with a handful of demo endpoints; also
  materially harder to explain per-prediction than Isolation Forest's
  path-length-based feature contributions.
- **Supervised classifier:** rejected outright — no reliable labeled
  attack data exists for this project, and fabricating labels would
  violate the "no fake AI/no fake security data" constraint.

## Consequences
Isolation Forest's explainability is approximate (feature contribution via
path length, not a formal SHAP-style guarantee) — documented as a known
limitation, not sold as full interpretability.
