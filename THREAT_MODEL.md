# Threat Model (STRIDE)

Scope: Endpoint Agent, Telemetry Channel, Ingestion API, Redis, PostgreSQL,
AI Engine, Response Engine, Frontend, Administrator Accounts, Tenant
Isolation, Update Mechanism.

Legend: **S**poofing, **T**ampering, **R**epudiation, **I**nfo disclosure,
**D**enial of service, **E**levation of privilege.

## Endpoint Agent

| Threat | STRIDE | Impact | Likelihood | Mitigation | Residual Risk |
|---|---|---|---|---|---|
| Attacker on the host tampers with/kills the agent to blind telemetry | T, D | High — loss of visibility | Medium | Heartbeat monitoring flags silent agents; missing-heartbeat itself is a detection signal | An attacker with root can still blind the agent — MVP does not claim tamper-proof kernel protection |
| Stolen agent credential used to inject fake telemetry | S, T | Medium — false telemetry poisons detection | Low-Medium | Credential scoped to one `agent_id`; revocable; anomalous telemetry patterns from a known agent are themselves flagged | Credential theft requires host compromise, which is already a bigger problem |
| Sensitive data over-collection | I | Medium — privacy/compliance | Low | Explicit data-minimization list; secret-flag redaction client-side | Residual: redaction is pattern-based and imperfect |

## Telemetry Channel

| Threat | STRIDE | Impact | Likelihood | Mitigation | Residual Risk |
|---|---|---|---|---|---|
| MITM on agent↔ingestion transport | S, T, I | High | Low (TLS) | TLS with certificate verification | Cert-pinning not yet implemented in MVP — documented gap |
| Replay of captured telemetry batch | S, R | Low-Medium | Low | `event_id` dedupe + timestamp window validation | — |
| Agent-declared `tenant_id`/`asset_id` spoofing | S, E | High — cross-tenant data injection | Medium if not mitigated | Ingestion derives tenant/asset from the authenticated credential, ignoring client-supplied identity fields | — |

## Ingestion API / Backend

| Threat | STRIDE | Impact | Likelihood | Mitigation | Residual Risk |
|---|---|---|---|---|---|
| Oversized/malformed payload DoS | D | Medium | Medium | Size caps, schema validation, rate limiting | — |
| Injection (SQL/NoSQL) via telemetry fields | T, E | High | Low | Parameterized queries via SQLAlchemy; `raw` JSON stored as JSONB, never interpolated into SQL | — |
| Broken object-level authorization (IDOR across tenants) | I, E | High | Medium if unchecked | Repository layer requires tenant context; RLS backstop | — |
| Verbose error responses leaking internals | I | Low-Medium | Medium if unhandled | Structured error responses, no stack traces to client | — |

## Redis

| Threat | STRIDE | Impact | Likelihood | Mitigation | Residual Risk |
|---|---|---|---|---|---|
| Unauthenticated Redis exposed on network | S, I, E | Critical | Low with proper config | `requirepass`, bound to internal Docker network only, not published externally | Misconfiguration risk remains a manual-review item |
| Stream flooding causing memory pressure | D | Medium | Medium | Stream max-length trimming (`MAXLEN ~`), consumer lag alerting | — |

## PostgreSQL

| Threat | STRIDE | Impact | Likelihood | Mitigation | Residual Risk |
|---|---|---|---|---|---|
| Cross-tenant data leak via bug in app code | I | High | Medium without RLS | Row-Level Security policies enforced at DB level | — |
| Credential compromise | S, E | High | Low | Least-privilege DB roles, secrets not in source control | — |

## AI Engine

| Threat | STRIDE | Impact | Likelihood | Mitigation | Residual Risk |
|---|---|---|---|---|---|
| Adversarial/gradual poisoning of baseline (slow-drift evasion) | T | Medium-High | Medium | Nightly retrain with bounded drift checks; analyst feedback loop offline, not live online-learning | Sophisticated slow attacks are a known, documented limitation — not solved in MVP |
| Over-trust in "AI verdict" | E (of trust) | Medium | Medium if UI is careless | Explainability requirement (top features + confidence always shown); AI never solely triggers response | — |

## Response Engine

| Threat | STRIDE | Impact | Likelihood | Mitigation | Residual Risk |
|---|---|---|---|---|---|
| Unauthorized/forged response action (e.g., malicious isolate request) | S, E | High — availability impact | Low | RBAC + approval gate for HIGH+, full audit trail | — |
| Response action abused for DoS against a legitimate asset | D | Medium | Low | Approval requirement above HIGH severity; every action requires a logged reason | — |

## Frontend / Dashboard

| Threat | STRIDE | Impact | Likelihood | Mitigation | Residual Risk |
|---|---|---|---|---|---|
| XSS via stored telemetry fields rendered in UI | T, I | Medium | Medium if unescaped | React's default escaping + explicit sanitization of any HTML-rendering paths | — |
| CSRF against state-changing actions | T | Medium | Low | JWT bearer auth (not cookie-based sessions) sidesteps classic CSRF; SameSite cookie config if refresh cookie is used | — |

## Administrator Accounts

| Threat | STRIDE | Impact | Likelihood | Mitigation | Residual Risk |
|---|---|---|---|---|---|
| Admin credential compromise (phishing, reuse) | S, E | Critical | Medium | MFA required for elevated roles, audit logging of all admin actions | Social engineering remains outside technical control |

## Update Mechanism (Agent)

| Threat | STRIDE | Impact | Likelihood | Mitigation | Residual Risk |
|---|---|---|---|---|---|
| Malicious config pushed to agent via compromised backend | T, E | Critical | Low | Config updates restricted to a fixed, versioned schema — no arbitrary command execution accepted from server | If the backend itself is compromised, this is a broader incident outside agent-level mitigation |

## Tenant Isolation (cross-cutting)

Covered above at each layer; the residual risk accepted for MVP is that
RLS + application filtering is a strong but not formally verified
guarantee — recommended future work is a dedicated cross-tenant fuzz/test
suite (see `docs/testing/testing-strategy.md`).
