# ADR-0008: Docker Compose for MVP deployment

## Status
Accepted

## Decision
Deploy via Docker Compose (frontend, backend, workers, Postgres, Redis,
Nginx) for all MVP phases.

## Alternatives Considered
- **Kubernetes:** the "expected" production topology for a platform this
  shape, but introduces cluster operations, manifests, and complexity with
  no scaling requirement that justifies it at MVP/demo scale. Explicitly
  rejected per project constraint (Section 23).

## Consequences
No built-in horizontal autoscaling or multi-node scheduling; acceptable
for a single-machine college demo. The modular-monolith boundary (ADR-0005)
is specifically kept clean so a future Kubernetes migration is additive,
not a redesign.
