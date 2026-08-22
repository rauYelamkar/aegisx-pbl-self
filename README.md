# AEGISX

**AI-Driven Autonomous Enterprise Security Service Platform**

AEGISX is a solo college PBL project implementing a realistic, working miniature
of a modern MSSP/SOC platform: agent-based telemetry collection, a hybrid
rules+ML detection pipeline, risk scoring, incident management, controlled
response/containment, and a multi-tenant SOC dashboard.

This is **Phase 0** of the project: specification and architecture only.
No application code has been implemented yet. See `docs/decisions/` for why
each major technology was chosen, and `AEGISX_MASTER_SPEC.md` for the
canonical requirements document.

## Status

| Phase | Name | Status |
|---|---|---|
| 0 | Specification + Architecture | **In progress (this document set)** |
| 1 | Repository + Infrastructure Foundation | Not started |
| 2 | Auth + Multi-Tenancy | Not started |
| 3 | Linux Agent + Low-Level Telemetry | Not started |
| 4 | Telemetry Ingestion + Event Pipeline | Not started |
| 5 | Detection + Correlation | Not started |
| 6 | AI/ML + Behavior Analysis | Not started |
| 7 | Risk + Incident Management | Not started |
| 8 | Response + Isolation + Remediation | Not started |
| 9 | SOC Dashboard | Not started |
| 10 | Security Architecture Builder | Not started |
| 11 | Attack Simulation | Not started |
| 12 | Security Hardening | Not started |
| 13 | Integration + Testing | Not started |
| 14 | Final PBL Release | Not started |

## Document Map

```text
README.md                                  You are here
AEGISX_MASTER_SPEC.md                       Product + functional/non-functional requirements
ARCHITECTURE.md                             System architecture, security layers, event flow
SECURITY.md                                 Security controls, AuthN/AuthZ, tenant isolation
THREAT_MODEL.md                             STRIDE threat model per trust boundary
MVP_SCOPE.md                                MVP vs future, technology validation
ROADMAP.md                                  14-phase implementation roadmap

docs/architecture/diagrams.md               Mermaid diagrams (system, layers, flows, ERD)
docs/architecture/database-schema.md        PostgreSQL schema
docs/api/api-specification.md               FastAPI v1 endpoint specification
docs/agent/agent-specification.md           Linux endpoint agent design
docs/ai/ai-ml-design.md                     AI/ML architecture, Isolation Forest, explainability
docs/detection/detection-methodology.md     Hybrid rules/statistics/ML/behavior detection
docs/response/response-model.md             Incident + response engine, state machines
docs/deployment/deployment-guide.md         Docker Compose topology
docs/testing/testing-strategy.md            Test strategy per layer
docs/decisions/                             Architecture Decision Records (ADR-0001 ... )
```

## Non-Goals

AEGISX does not attempt to replace or clone CrowdStrike, Cortex, Defender, or
Splunk. It does not claim complete EDR/SIEM/SOAR coverage, complete kernel
protection, or production-grade autonomous response. Every capability listed
as "MVP" in `MVP_SCOPE.md` is intended to be genuinely functional; everything
else is explicitly marked future scope or simulated.
