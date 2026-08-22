# Testing Strategy

Testing is executed in dedicated phases (Phase 13), but every component is
written to be testable from the start (dependency injection for
DB/Redis clients, no hidden global state, deterministic risk-engine logic).

## Unit

- **Backend:** `pytest` + `pytest-asyncio` for services (risk engine
  scoring, rule evaluation, state machine transitions, RBAC permission
  checks) and schema validation.
- **Frontend:** `Vitest` + `React Testing Library` for components and
  dashboard state (loading/empty/error/degraded states explicitly tested).

## Integration

- `httpx.AsyncClient` against a real (test-container) Postgres + Redis for
  API endpoint tests, including a **negative cross-tenant test**: user
  from Org A must never retrieve Org B's assets/incidents (validates RLS +
  application-layer isolation together).
- Agent registration/heartbeat flow tested against a running ingestion API.

## End-to-End

`Playwright` drives the full lifecycle from `AEGISX_MASTER_SPEC.md` §2:
register client → generate architecture → register asset → start agent
(or a scripted telemetry replay) → run attack simulation → verify incident
appears with correct MITRE mapping and risk score → execute containment →
verify recovery → generate report.

## Load

`Locust` exercises the ingestion API at a multiple of expected agent
volume to validate backpressure (429 + agent backoff) rather than silent
data loss.

## Security Testing

| Tool | Purpose | CI Stage |
|---|---|---|
| Semgrep | SAST across Python/TypeScript | Every PR |
| Bandit | Python-specific security lint | Every PR |
| Trivy | Container image + dependency vulnerability scan | Every PR / nightly |
| Gitleaks | Secret scanning | Every PR (pre-commit too) |
| Dependabot | Automated dependency update PRs | Continuous |
| OWASP ZAP | DAST baseline scan against a running staging instance | Nightly / pre-release |

CI fails the build on high-severity SAST/dependency findings; findings are
triaged, not silently suppressed.
