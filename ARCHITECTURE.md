# AEGISX System Architecture

Status: Phase 0. This document is the primary architecture reference;
diagrams live in `docs/architecture/diagrams.md`.

## 1. High-Level Architecture

```text
CLIENT ENVIRONMENT
      │
      ├── Endpoint (Linux) ── AEGISX Agent ── eBPF / Auditd / procfs
      │
      └── (future) Server sensor / Network sensor
                │
        Secure Transport (mTLS-capable HTTPS, agent JWT)
                │
                ▼
       TELEMETRY INGESTION (FastAPI)
                │  validate → normalize
                ▼
          REDIS STREAMS (per-tenant partitioned streams)
                │
                ▼
        EVENT PROCESSING (consumer group)
        ┌───────────┼────────────┐
        │           │            │
     Rules      Statistics /   Threat Intel
                   ML (Isolation Forest)
        └───────────┼────────────┘
                     ▼
              CORRELATION (behavior grouping)
                     ▼
                RISK ENGINE (deterministic, weighted)
                     ▼
              INCIDENT ENGINE (state machine + MITRE mapping)
                     ▼
           RESPONSE ORCHESTRATOR (policy-gated)
        ┌───────────┼────────────┐
        │           │            │
     Isolate      Block       Remediate
        └───────────┼────────────┘
                     ▼
                  RECOVERY
                     ▼
              SOC DASHBOARD (Next.js) ── Client / Analyst
```

## 2. Component Responsibility Matrix

| Component | Responsibility | Inputs | Outputs | Trust Boundary | Failure Behavior |
|---|---|---|---|---|---|
| Agent | Collect + buffer + ship telemetry | Kernel/OS events | Signed telemetry batches | Untrusted (compromised host) | Local buffer + retry with backoff; never blocks host operation |
| Ingestion API | AuthN agent, validate schema, enqueue | HTTPS telemetry batches | Redis Stream entries | Semi-trusted (authenticated agent) | Reject malformed/oversized payloads; 4xx with no internal detail leaked |
| Event Processor | Consume stream, run rules/stats/ML, emit findings | Normalized events | Findings (rule hits, anomaly scores) | Trusted | At-least-once processing; idempotent by `event_id` |
| Correlation Service | Group related findings into behavior narratives | Findings | Correlated behavior clusters | Trusted | Falls back to per-event handling if correlation window times out |
| Risk Engine | Deterministic weighted scoring | Findings + asset criticality + intel + history | Risk score (0–100) + explanation | Trusted | Never silently drops a signal; missing inputs reduce confidence, not silently zero the score |
| Incident Engine | State machine, MITRE mapping, evidence assembly | Risk score ≥ threshold | Incident record | Trusted | Duplicate suppression via correlation key |
| Response Orchestrator | Policy evaluation, approval gate, action dispatch | Incident + policy | Response action + audit log | Trusted, high-privilege | HIGH+ requires human approval unless policy explicitly allows auto-contain |
| SOC Dashboard | Present real backend data | API reads + WebSocket/poll | Analyst actions | Trusted client, RBAC-scoped | Loading/empty/error/degraded states; never fabricates metrics |

## 3. Layered Security Model (Summary)

Full detail per layer is in `docs/architecture/diagrams.md` (Layer diagram)
and `MVP_SCOPE.md` (implementation status). Summary:

| Layer | Scope | MVP Status |
|---|---|---|
| 8 — Cloud / External | External-facing infra, DNS, exposure | Architecture only |
| 7 — Identity / Access | Auth, RBAC, sessions | **Implemented** |
| 6 — Applications / APIs | AEGISX's own API surface | **Implemented** |
| 5 — Servers / Databases | Server-class asset monitoring | Partial (same agent, server role tag) |
| 4 — Network | Connection/flow telemetry | **Implemented** (agent-observed only, no dedicated NIDS) |
| 3 — Employee Endpoints | Linux workstation/server telemetry | **Implemented** |
| 2 — Operating System | Process, file, auth events | **Implemented** |
| 1 — Kernel / Low-Level | eBPF/audit-sourced telemetry | **Implemented** (subset — see Agent spec) |

## 4. Kernel / Low-Level Architecture

```text
Kernel
  ↓ (execve, connect, open/unlink syscalls; auditd rules)
eBPF programs (bcc/libbpf-based) + Linux Audit (auditd + auditctl rules)
  ↓
Agent collector (reads eBPF ring buffer / audit log, no kernel module written)
  ↓
Normalized telemetry event
```

Design decisions:

- **No custom kernel module.** eBPF via existing verified hooks (execve,
  connect, file open/unlink) plus `auditd` rules cover process, network, and
  file-integrity signals without kernel-module risk or maintenance burden.
- **procfs/inotify fallback.** On systems where eBPF cannot be loaded
  (missing capability, kernel too old, container without `CAP_BPF`), the
  agent degrades to `/proc` polling + `inotify`/`fanotify`, and marks its
  own telemetry as "degraded source" so downstream detection can lower
  confidence accordingly instead of silently pretending it's kernel-grade.
- **Security boundary:** eBPF programs run in a restricted, verified
  sandbox inside the kernel; the agent process itself runs as a
  system service requiring elevated privileges (`CAP_BPF`/`CAP_AUDIT_READ`)
  documented explicitly in the agent spec's threat model.
- AEGISX does **not** claim complete kernel protection — this is real,
  bounded low-level telemetry, not a full kernel security product.

## 5. Endpoint Agent (Summary)

Full spec: `docs/agent/agent-specification.md`. Agent is a Python service
(MVP) with a lifecycle: install → register (mTLS/JWT enrollment) → heartbeat
→ collect → locally buffer (SQLite/disk queue) → ship in batches → retry on
failure with exponential backoff → apply config updates on heartbeat
response.

## 6. Telemetry Event Model

Canonical envelope (all event types share this shape; `data` is
type-specific):

```json
{
  "event_id": "uuid",
  "schema_version": "1.0",
  "tenant_id": "uuid",
  "agent_id": "uuid",
  "asset_id": "uuid",
  "timestamp": "2026-08-22T10:00:00Z",
  "event_type": "process_start | process_end | network_connection | auth_success | auth_failure | file_create | file_modify | file_delete | system_health | agent_heartbeat",
  "source": "ebpf | auditd | procfs | inotify | agent",
  "severity": "info | low | medium | high | critical",
  "correlation_id": "uuid | null",
  "data": { "...event-type-specific fields..." }
}
```

Rules: `tenant_id`/`agent_id` in the payload are **never** trusted as-is —
ingestion re-derives them from the authenticated agent's credential and
rejects any mismatch (see `SECURITY.md`).

## 7. Event Pipeline

```text
Agent → HTTPS POST /api/v1/telemetry (batched, gzip) → FastAPI validation
  → normalization → XADD to tenant-scoped Redis Stream
  → consumer group (`detection-workers`) → per-event processing
  → findings written to Postgres + published to a `findings` stream
  → correlation/risk/incident consumers
```

- **Idempotency:** consumers dedupe on `event_id`; reprocessing a delivered-
  but-unacked entry is safe.
- **Ordering:** ordering is only guaranteed per-asset (single stream key per
  asset), not globally — acceptable for MVP detection logic, documented as a
  known limitation.
- **Backpressure:** ingestion returns HTTP 429 if the stream's pending-entries
  list exceeds a configured threshold; agent backs off and buffers locally.
- **Dead-letter:** entries that fail processing 3× move to a
  `*:deadletter` stream for manual/automated inspection, never silently
  dropped.

## 8. Detection, AI, Risk, Incident, Response

Detailed in their own documents:
`docs/detection/detection-methodology.md`, `docs/ai/ai-ml-design.md`,
`docs/response/response-model.md` (covers both incident and response
engines).

## 9. Security Zones & Containment

```text
Internet / Untrusted
   │
Employee Zone ──┐
Server Zone ─────┼── AEGISX-modeled security zones (logical, per tenant)
App Zone ────────┤
DB Zone ─────────┤
Management Zone ─┘
   │
Quarantine Zone  ← isolated assets moved here on containment
```

Containment is **localized**: isolating one asset moves only that asset's
logical zone membership to Quarantine (enforced via response actions such
as `ISOLATE_ENDPOINT`/`BLOCK_IP`, executed against the lab environment or
simulated for out-of-lab assets). The rest of the tenant's assets and zones
are unaffected.

## 10. MSSP / Multi-Tenancy

Every tenant-scoped table carries `organization_id`. Enforcement happens at
three layers, defense-in-depth style:

1. **API layer:** JWT claims carry `organization_id` + role; every request
   handler injects it into the query context — it is never read from the
   request body/path for authorization purposes.
2. **Service layer:** repository methods require an explicit tenant context
   object; there is no "list all" method without a tenant filter.
3. **Database layer:** composite indexes/foreign keys keyed by
   `organization_id`; Postgres Row-Level Security (RLS) policies as a final
   backstop so a bug in application code cannot leak cross-tenant rows.

Cross-tenant access (MSSP_ADMIN/SUPER_ADMIN) is an explicit, separately
audited permission — not the default for any role.
