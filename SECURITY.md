# Security Architecture

## Authentication

- User auth: email + password (Argon2id hashing), JWT access token
  (short TTL, ~15 min) + refresh token (rotated, revocable, stored
  hashed server-side), optional TOTP MFA.
- Agent auth: one-time enrollment token → long-lived agent JWT, rotated
  periodically; agent credential is scoped to a single `agent_id`/`asset_id`
  and cannot impersonate another agent.
- No session state trusted from the client beyond the signed JWT claims;
  `organization_id` and `role` come from the token, never from a request
  body/query parameter.

## Authorization (RBAC)

- Enforced server-side via a FastAPI dependency
  (`require_permission("<resource>:<action>")`) on every protected route.
- Roles: `SUPER_ADMIN, MSSP_ADMIN, SOC_ANALYST, SECURITY_ENGINEER,
  INCIDENT_RESPONDER, CLIENT_ADMIN, VIEWER` — permissions are explicit
  grants (`role_permissions` table), not inferred from role name strings.
- Frontend hides unavailable actions for UX only; it is never the
  authorization boundary.

## Tenant Isolation

Defense in depth (see `ARCHITECTURE.md` §10):
1. JWT-derived `organization_id` injected into every query context.
2. Repository layer has no un-scoped "list all" method.
3. Postgres Row-Level Security policies as a backstop against application bugs.

## Transport & Secrets

- TLS everywhere (agent↔ingestion, browser↔API, internal service calls
  where the deployment topology crosses a network boundary).
- Secrets (DB password, JWT signing key, Redis auth) come from environment
  variables / Docker secrets, never committed; `.env.example` documents
  required variables, `.env` is gitignored.
- Passwords are never logged; command-line arguments matching known
  secret-flag patterns are redacted client-side by the agent before
  transmission.

## Input/Output Validation

- All API input validated against explicit Pydantic schemas; unknown
  fields rejected, not silently ignored.
- Telemetry payloads are size-capped and schema-versioned; malformed
  batches are rejected with a structured error, not partially processed.
- API responses use dedicated response models — ORM/database objects are
  never serialized directly, preventing accidental field leakage.

## Rate Limiting & Abuse Prevention

- Auth endpoints: per-IP and per-account rate limiting with backoff
  (brute-force protection).
- Telemetry ingestion: per-agent rate limiting tied to expected event
  volume; excess triggers 429 + agent-side backoff, not silent drops.

## Audit Logging

- `audit_logs` is append-only (no UPDATE/DELETE grants for the
  application role) and records every authentication event, permission
  change, response action, and incident state transition with actor,
  target, timestamp, and metadata.

## Least Privilege

- Agent host process runs with only the specific Linux capabilities it
  needs (`CAP_BPF`, `CAP_AUDIT_READ`, etc.), not root.
- Database roles are split: the API's DB role cannot `TRUNCATE`/`DROP`;
  a separate migration role is used only during deploys.
- Response executors that perform containment actions run under a
  distinct, narrowly-scoped credential from the general API service.

## Dependency & Static Analysis (planned tooling, per `docs/testing/testing-strategy.md`)

Semgrep, Bandit, Trivy, Gitleaks, Dependabot, and OWASP ZAP are run in
CI/CD (Phase 12 — Security Hardening) — not adopted merely to inflate the
tool count; each maps to a specific control (SAST, container scanning,
secret scanning, dependency updates, DAST).

## Explicit Non-Guarantees

AEGISX does not claim: complete kernel protection, complete EDR/SIEM/SOAR
coverage, or safety against a sufficiently privileged/adaptive attacker
targeting the agent host itself. These limitations are documented, not
hidden — see `THREAT_MODEL.md` for residual risk per component.
