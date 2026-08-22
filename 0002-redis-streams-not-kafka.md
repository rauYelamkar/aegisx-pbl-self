# ADR-0002: Redis Streams instead of Kafka for the event pipeline

## Status
Accepted

## Context
Telemetry needs a durable, ordered-per-partition, at-least-once event
transport between ingestion and detection workers.

## Decision
Use Redis Streams with consumer groups, tenant/asset-scoped stream keys,
and `MAXLEN ~` trimming for retention. Redis is already required for
caching/rate-limiting, so this avoids introducing a second messaging
system.

## Alternatives Considered
- **Kafka:** industry standard for this pattern at scale, but operationally
  heavy (ZooKeeper/KRaft, partition management, broker ops) for a
  solo-developer MVP with no throughput requirement that justifies it.
  Explicitly rejected per the project's engineering constraint
  (Section 23 of the project context: avoid unnecessary complexity).

## Consequences
Redis Streams' delivery/ordering guarantees are weaker than Kafka's at
very large scale; acceptable for MVP volume. Revisit if a genuine
multi-hundred-agent throughput requirement emerges — documented as a
future migration candidate, not a blocker.
