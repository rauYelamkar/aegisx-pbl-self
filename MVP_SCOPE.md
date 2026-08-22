# MVP Scope, Technology Validation, and Future Roadmap

## MVP vs Future

| Capability | MVP | Future |
|---|---|---|
| Linux agent | Yes | Advanced (perf tuning, broader syscall coverage) |
| Low-level telemetry | Yes (eBPF + audit, with documented fallback) | Deeper kernel coverage, LSM hooks |
| Windows/macOS agent | No | Yes |
| Network detection | Basic (agent-observed connections only) | Dedicated NIDS/flow sensor |
| Cloud security (Layer 8) | Architecture/simulation only | Real AWS/Azure/GCP API integration |
| AI anomaly detection | Yes (Isolation Forest, per-org) | Advanced models, per-asset-class models |
| Autonomous response | Controlled lab, policy-gated, approval above HIGH | Production-grade, broader action catalog |
| MSSP multi-tenancy | Yes (RLS + app-layer isolation) | Enterprise-scale sharding/partitioning |
| Threat intelligence | Basic/seeded indicators | Live commercial feed integration |
| Full SIEM | No | Future |
| Full EDR | No | Future |
| Full SOAR | No | Future (Response Engine is a bounded subset) |

## Technology Validation

| Technology | Chosen because | Rejected alternative | Why rejected |
|---|---|---|---|
| FastAPI + Pydantic | Async, typed contracts, auto OpenAPI, fast to build correctly | Django REST Framework | Heavier, less natural async story for this workload |
| PostgreSQL | Relational integrity for tenant/RBAC/incident data, RLS support | MongoDB | Weaker for the highly relational RBAC/tenant/incident model; JSONB gives flexibility where genuinely needed |
| Redis Streams | Lightweight, consumer groups give at-least-once semantics, already needed for caching/rate-limiting | Kafka | Operationally heavy for a solo-developer MVP; no throughput requirement justifies it (see `docs/decisions/0003-redis-streams.md`) |
| Python agent | Fastest path to a correct, testable cross-cutting agent; team already Python-heavy elsewhere | Go agent | Would improve resource footprint but doubles the language surface for a solo developer; documented as a future migration candidate if performance demands it |
| eBPF + auditd | Real low-level telemetry without a custom kernel module | Custom kernel module | Unjustified risk/maintenance burden for MVP; explicitly rejected per project constraints |
| Docker Compose | Simple, sufficient for MVP scale | Kubernetes | No horizontal-scaling requirement exists yet; adopting k8s now is complexity without benefit |
| Isolation Forest | Unsupervised, cheap, works without labeled attack data | Deep autoencoder | Needs more data/GPU infra than justified at MVP scale |
| Modular monolith | One deployable unit, clear internal service boundaries, can be split later | Microservices | Dozens of services for a solo developer would slow delivery without a scaling need that justifies it |

## Engineering Constraint Reminder

Architectural sophistication must not become unnecessary complexity. Every
"Future" row above is a deliberate scope cut, not an oversight — see
`docs/decisions/` for the full reasoning behind each accepted technology.
