# ADR-0001: FastAPI + PostgreSQL + SQLAlchemy as the primary backend

## Status
Accepted

## Context
The MVP needs a typed, async-capable API layer over a highly relational
domain (organizations, RBAC, assets, incidents, audit) plus a large volume
of semi-structured telemetry.

## Decision
Use FastAPI + Pydantic for the API layer, SQLAlchemy (async) + PostgreSQL
for persistence. Telemetry payloads store a JSONB `raw` column alongside
typed specialization tables for the fields that need relational querying.

## Alternatives Considered
- **Django REST Framework:** more batteries-included but heavier, weaker
  native async story, and its ORM patterns don't map as cleanly onto
  Redis Streams consumer workers.
- **MongoDB for everything:** would simplify telemetry storage but weakens
  the RBAC/tenant/incident relational integrity that is core to this
  product's correctness requirements (least favorable place to lose
  referential integrity).

## Consequences
Requires disciplined schema/migration management (Alembic) as the domain
grows; gains strong typing, OpenAPI-driven contracts, and RLS-based tenant
isolation "for free" from Postgres.
