# Deployment Architecture (MVP)

## Topology (Docker Compose)

```text
Nginx (reverse proxy, TLS termination)
  ├── / (Next.js frontend)
  ├── /api (FastAPI backend)
  └── /agent-ingest (FastAPI, same service, dedicated rate-limit zone)

FastAPI Backend
  ├── depends on: PostgreSQL, Redis

Detection/AI Workers (separate container, same codebase, worker entrypoint)
  ├── depends on: PostgreSQL, Redis

PostgreSQL (persistent volume)
Redis (persistent volume for stream durability, AOF enabled)
```

## Services (`docker-compose.yml`, conceptual)

| Service | Image basis | Notes |
|---|---|---|
| `frontend` | Node 20 (Next.js build) | Served via Nginx in prod mode |
| `backend` | Python 3.12 slim | FastAPI + Uvicorn/Gunicorn workers |
| `worker` | Same image as backend | Different entrypoint: consumer group processor |
| `postgres` | `postgres:16` | Named volume, RLS-enabled schema |
| `redis` | `redis:7` | `requirepass` set, AOF persistence, internal network only |
| `nginx` | `nginx:alpine` | TLS termination, reverse proxy, rate-limit zones |

## Environment Configuration

`.env.example` documents (values never committed):

```text
DATABASE_URL=
REDIS_URL=
JWT_SECRET=
JWT_REFRESH_SECRET=
AGENT_ENROLLMENT_SIGNING_KEY=
TLS_CERT_PATH=
TLS_KEY_PATH=
```

## Health & Readiness

- `/healthz` (liveness) and `/readyz` (readiness — checks DB/Redis
  connectivity) on the backend.
- Workers expose a lightweight metrics/health port for consumer-lag
  monitoring.

## Observability (MVP)

- Structured JSON logs to stdout (collected by the container runtime;
  no dedicated log aggregation service required for MVP demo).
- Correlation/request IDs propagated from Nginx → backend → worker logs.
- Basic Prometheus-format `/metrics` endpoint on backend and workers
  (request counts/latency, consumer lag, ingestion throughput).

## Scaling Notes (documented, not built for MVP)

The `worker` service can be scaled horizontally (`docker compose up --scale
worker=N`) since Redis consumer groups distribute stream entries across
consumers — this is validated as a design property even though the MVP
demo runs a single worker replica.
