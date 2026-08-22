# AEGISX Master Specification

Status: Phase 0 — Living document, updated as architectural decisions change.

## 1. Product

**Name:** AEGISX
**Full name:** AI-Driven Autonomous Enterprise Security Service Platform

**Description:** AEGISX is a multi-tenant security operations platform that
lets a client organization register its infrastructure, deploy a Linux
telemetry agent, and get continuous monitoring, hybrid detection (rules +
statistics + ML), risk-scored incidents, MITRE-mapped analysis, and policy-
gated response/containment — presented through a real-time SOC dashboard.

**Vision:** Demonstrate, at achievable solo-developer MVP scale, the same
architectural pattern used by production MSSP/EDR/SIEM platforms: telemetry
from the lowest practical layer (kernel/OS) flowing upward through detection,
correlation, risk, incident, and response stages, with tenant isolation and
auditability at every step.

**Problem statement:** Small/mid organizations cannot afford or operate a
full SOC. Commercial EDR/SIEM/SOAR platforms are opaque, expensive, and
architecturally inaccessible for learning. AEGISX demonstrates a transparent,
explainable, defensible version of that pipeline end-to-end.

**Target users (personas):**

| Persona | Role | Needs |
|---|---|---|
| SOC Analyst | Investigates incidents | Real-time telemetry, incident timeline, evidence |
| Security Engineer | Tunes detection/policy | Rule config, risk thresholds, MITRE mapping |
| Incident Responder | Executes containment | Isolation/remediation actions, audit trail |
| Client Admin | Owns the tenant | Asset/user management, reports |
| MSSP Admin | Manages multiple clients | Cross-tenant oversight (authorized only) |
| Super Admin | Platform owner | Global configuration, tenant provisioning |
| Viewer | Read-only stakeholder | Dashboards, reports |

**Target organizations:** Single small/mid-size org for MVP demo; architecture
supports multiple tenants (MSSP model) from day one even though the demo
uses 1–3 seeded tenants.

## 2. MVP Objectives

The MVP must prove the complete lifecycle end-to-end, on real data, for at
least one Linux endpoint:

```text
Register Client → Build Architecture → Register Asset → Deploy Agent →
Collect Real Telemetry → Establish Baseline → Run Controlled Attack
Simulation → Detect → AI Anomaly Score → Risk Score → Create Incident →
MITRE Map → Contain (Isolate) → Investigate → Remediate → Verify → Recover →
Report
```

This flow is the Definition of Done for the MVP as a whole (see
`ROADMAP.md`, Phase 14).

## 3. Functional Requirements

| ID | Requirement | MVP |
|---|---|---|
| FR-1 | Org can register and manage users under RBAC | Yes |
| FR-2 | Org can describe/register assets (endpoints) manually | Yes |
| FR-3 | Org can generate/edit a visual security architecture (React Flow) | Yes |
| FR-4 | Agent installs, registers, authenticates, and streams telemetry from a Linux host | Yes |
| FR-5 | Telemetry covers process, network, auth, filesystem, and system/health events | Yes |
| FR-6 | Low-level telemetry sourced from eBPF and/or Linux Audit, not just polling | Yes |
| FR-7 | Ingestion validates and normalizes events into a canonical schema | Yes |
| FR-8 | Events flow through Redis Streams to independent consumers | Yes |
| FR-9 | Deterministic rule engine flags known-bad patterns | Yes |
| FR-10 | Statistical baseline + Isolation Forest flags anomalies | Yes |
| FR-11 | Behavior correlation groups related events into a narrative | Yes |
| FR-12 | Risk engine combines signals into an explainable 0–100 score | Yes |
| FR-13 | Incidents are auto-created above a risk threshold, with full state machine | Yes |
| FR-14 | Incidents map to MITRE ATT&CK techniques | Yes |
| FR-15 | Policy-gated response: isolate endpoint, block IP, terminate process, quarantine file | Yes |
| FR-16 | All response actions are authenticated, authorized, and audit-logged | Yes |
| FR-17 | Controlled attack simulator injects synthetic events through the real pipeline | Yes |
| FR-18 | SOC dashboard shows live assets, incidents, risk, telemetry, MITRE mapping | Yes |
| FR-19 | Incident reports are generated from real incident data | Yes |
| FR-20 | Multi-tenant data isolation enforced at API/service/DB layers | Yes |
| FR-21 | Windows/macOS agents | Future |
| FR-22 | Live threat-intel feed integration | Future (static/seeded intel for MVP) |
| FR-23 | Fully autonomous (no-approval) response for all severities | Future — MVP requires approval above HIGH |

## 4. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Security | TLS everywhere, hashed credentials (Argon2/bcrypt), JWT-based auth, RBAC enforced server-side, tenant isolation enforced at query layer, no secrets in source control |
| Reliability | Ingestion tolerates agent disconnect/retry; Redis consumer groups provide at-least-once processing with idempotent handlers |
| Performance | Ingestion API must sustain the demo load (single-digit endpoints, bursts during simulation) with p95 ingest latency < 500ms |
| Scalability | Modular monolith with clean service boundaries so components can be split into services later without a rewrite |
| Observability | Structured JSON logs, correlation IDs, health/readiness endpoints, basic Prometheus metrics |
| Maintainability | Explicit Pydantic/OpenAPI contracts; no direct ORM model leakage into API responses |
| Privacy | Data minimization on the agent; no unrelated personal data collected |
| Auditability | Every state transition (incident, response action) is logged with actor, timestamp, reason |

## 5. Out of Scope for MVP

- Windows/macOS agents
- Real threat-intel feed subscriptions
- Kubernetes, Kafka, service mesh
- Fully unsupervised autonomous response at CRITICAL severity without any policy gate
- Custom kernel modules
- Cloud-provider-specific security integrations (AWS/Azure/GCP APIs) beyond architectural placeholders

See `MVP_SCOPE.md` for the full MVP-vs-future matrix and `docs/decisions/`
for the reasoning behind each major technology choice.
