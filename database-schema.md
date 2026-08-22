# Database Architecture (PostgreSQL)

Modular monolith, single Postgres instance for MVP. Every tenant-scoped
table has `organization_id UUID NOT NULL REFERENCES organizations(id)` and a
Row-Level Security policy `USING (organization_id = current_setting('app.current_org')::uuid)`
as a defense-in-depth backstop behind application-layer filtering.

## Core Entities

| Table | Key Columns | Notes |
|---|---|---|
| `organizations` | id, name, tier, created_at | Tenant root |
| `users` | id, organization_id, email (unique per org), password_hash, mfa_enabled | Argon2id hashing |
| `roles` | id, name (enum: SUPER_ADMIN, MSSP_ADMIN, SOC_ANALYST, SECURITY_ENGINEER, INCIDENT_RESPONDER, CLIENT_ADMIN, VIEWER) | |
| `user_roles` | user_id, role_id, organization_id | Many-to-many, tenant-scoped |
| `permissions` | id, code, description | e.g. `incident:contain`, `asset:isolate` |
| `role_permissions` | role_id, permission_id | |
| `assets` | id, organization_id, name, type (endpoint/server/db/network/app/cloud), criticality (1-5), zone_id | |
| `security_zones` | id, organization_id, name, type (employee/server/db/app/mgmt/quarantine), trust_level | |
| `agents` | id, organization_id, asset_id, version, status (active/degraded/offline), last_heartbeat, enrollment_token_id | |
| `telemetry_events` | id (event_id), organization_id, agent_id, asset_id, timestamp, event_type, source, severity, schema_version, correlation_id, raw jsonb | Partitioned by month; append-only |
| `process_events` | event_id FK, pid, ppid, executable, cmdline, user, exit_code | 1:1 specialization |
| `network_events` | event_id FK, src_ip, dst_ip, src_port, dst_port, protocol, dns_query | |
| `authentication_events` | event_id FK, user, result, auth_source, source_ip | |
| `file_events` | event_id FK, path, action, hash, actor_pid | |
| `threat_indicators` | id, organization_id (nullable = global), type (ip/domain/hash), value, source, confidence | Indexed for lookup joins |
| `ai_predictions` | id, event_id FK, model_name, model_version, anomaly_score, confidence, top_features jsonb | |
| `risk_scores` | id, organization_id, asset_id, score, severity, components jsonb, computed_at | `components` stores explainability breakdown |
| `incidents` | id, organization_id, title, status (enum, see lifecycle), severity, risk_score_id, assigned_to, mitre_techniques text[], created_at, closed_at | |
| `incident_events` | id, incident_id, event_id FK nullable, note, actor, created_at | Timeline entries |
| `response_actions` | id, incident_id, action_type (enum), status (pending/approved/executing/succeeded/failed/rolled_back), target_asset_id, actor, reason, requires_approval bool, approved_by, executed_at | |
| `remediation_actions` | id, incident_id, action_type, status, verified_at | |
| `security_policies` | id, organization_id, name, risk_thresholds jsonb, auto_response_rules jsonb, enabled | |
| `audit_logs` | id, organization_id, actor_id, actor_type (user/agent/system), action, target_type, target_id, metadata jsonb, created_at | Append-only, never updated/deleted |

## Indexing & Constraints

- `telemetry_events`: composite index `(organization_id, asset_id, timestamp DESC)`; monthly range partitions for retention/performance.
- `incidents`: index `(organization_id, status, severity)` for dashboard queries.
- `users.email`: unique constraint scoped `(organization_id, email)`.
- All FKs `ON DELETE RESTRICT` for audit-relevant tables (`audit_logs`, `incidents`, `response_actions`); `ON DELETE CASCADE` only for pure child rows (`process_events` etc. cascading from `telemetry_events`).
- UUID primary keys (`gen_random_uuid()`) throughout for safe cross-service references and non-guessable IDs.

## Retention (MVP defaults, configurable per org)

| Data | Retention |
|---|---|
| Raw telemetry_events | 30 days hot, then archived/dropped |
| Findings / AI predictions | 90 days |
| Incidents + timeline | Indefinite (until org deletion) |
| Audit logs | 1 year minimum |
| Agent heartbeats | 7 days |

Not over-normalized: `telemetry_events.raw` keeps the full normalized JSON
so that new detection logic can be replayed against history without a
schema migration for every new field; typed specialization tables exist
only where structured querying (joins, aggregates) is actually needed.
