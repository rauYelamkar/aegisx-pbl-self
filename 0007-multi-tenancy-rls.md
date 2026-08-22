# ADR-0007: Row-Level Security as tenant-isolation backstop

## Status
Accepted

## Decision
Enforce tenant isolation at three layers: JWT-derived context at the API,
mandatory tenant-scoped repository methods at the service layer, and
Postgres Row-Level Security policies at the database layer, so that a bug
in any single layer does not by itself leak cross-tenant data.

## Alternatives Considered
- **Database-per-tenant:** strongest isolation, but operationally heavy
  (migration fan-out, connection pool sizing) for MVP tenant counts (1-3
  demo orgs). Documented as a future option if enterprise-scale tenancy is
  ever required (see `MVP_SCOPE.md`).
- **Application-layer filtering only (no RLS):** rejected — a single missed
  `WHERE organization_id = ...` clause becomes a critical cross-tenant
  data leak with no backstop.

## Consequences
RLS requires setting `app.current_org` per request/session and adds a
small amount of connection-handling complexity (session variable must be
set per request under a pooled connection), documented in
`docs/deployment/deployment-guide.md`.
