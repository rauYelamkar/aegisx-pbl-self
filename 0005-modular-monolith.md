# ADR-0005: Modular monolith over microservices

## Status
Accepted

## Context
AEGISX has many conceptual services (ingestion, detection, AI, risk,
incident, response) but is built and operated by one developer.

## Decision
Build a single deployable backend with clear internal module boundaries
(separate packages per domain: `telemetry/`, `detection/`, `ai/`, `risk/`,
`incidents/`, `response/`), each with its own service interface, so it can
be split into real microservices later without a rewrite — but is not
split prematurely.

## Alternatives Considered
- **Microservices from day one:** aligns with "how production MSSPs are
  built" aesthetically, but introduces network-boundary complexity,
  deployment overhead, and distributed-transaction concerns with no
  corresponding scaling need at MVP stage. Explicitly rejected per project
  constraint (Section 23: prefer modular monolith where appropriate).

## Consequences
Some internal calls that would be network calls in a microservices
architecture are in-process function calls for now; interfaces are kept
service-shaped (explicit request/response objects, no shared mutable
state) specifically so a future extraction is mechanical, not a rewrite.
