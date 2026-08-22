# Development Roadmap

```text
PHASE 0  — Specification + Architecture               (this document set)
PHASE 1  — Repository + Infrastructure Foundation      (Docker Compose skeleton, CI scaffolding)
PHASE 2  — Authentication + Multi-Tenancy Foundation    (users, roles, JWT, RLS)
PHASE 3  — Linux Agent + Low-Level Telemetry            (eBPF/audit collectors, enrollment)
PHASE 4  — Telemetry Ingestion + Event Pipeline         (FastAPI ingest, Redis Streams)
PHASE 5  — Detection + Correlation                      (rule engine, behavior clustering)
PHASE 6  — AI/ML + Behavior Analysis                    (feature extraction, Isolation Forest)
PHASE 7  — Risk + Incident Management                   (risk engine, incident state machine)
PHASE 8  — Response + Isolation + Remediation            (policy engine, response orchestrator)
PHASE 9  — SOC Dashboard                                 (Next.js, real backend data)
PHASE 10 — Security Architecture Builder                 (React Flow visual model)
PHASE 11 — Attack Simulation                              (synthetic events through real pipeline)
PHASE 12 — Security Hardening                             (Semgrep/Bandit/Trivy/Gitleaks/ZAP in CI)
PHASE 13 — Integration + Testing                          (E2E: agent → telemetry → incident → response)
PHASE 14 — Final PBL Release                              (docs, demo script, known limitations)
```

## Phase Exit Criteria (summary)

| Phase | Exit Criteria |
|---|---|
| 1 | `docker compose up` brings up empty FastAPI + Next.js + Postgres + Redis + Nginx skeleton with health checks passing |
| 2 | Login, RBAC enforcement, and tenant isolation (RLS) proven with a cross-tenant negative test |
| 3 | Agent enrolls, heartbeats, and ships at least process/network/auth/file events from a real Linux host |
| 4 | Telemetry visible end-to-end from agent through Redis Streams into Postgres, with idempotent replay |
| 5 | At least 3 rules + statistical baseline produce findings on real and simulated data |
| 6 | Isolation Forest trained on real baseline data, predictions stored with explainability fields |
| 7 | Risk score computed from real components; incidents auto-created and traverse the full state machine |
| 8 | Policy-gated response executes `ISOLATE_ENDPOINT` against a lab asset with full audit trail |
| 9 | Dashboard shows live assets/incidents/risk with no hardcoded metrics |
| 10 | Architecture builder renders a tenant's assets/zones as an editable React Flow graph |
| 11 | Simulated attack chain produces a real incident through the unmodified real pipeline |
| 12 | CI pipeline runs security tooling and fails the build on high-severity findings |
| 13 | Automated E2E test covers the Section 25 demonstration flow from `AEGISX_MASTER_SPEC.md` |
| 14 | Full demonstration flow runs live; known limitations documented, no capability overclaimed |

This roadmap will be updated if a phase reveals a genuine need to revise
an earlier architectural decision — any such change is recorded as a new
ADR in `docs/decisions/`, not made silently.
