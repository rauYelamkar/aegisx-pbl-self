# ADR-0003: Python for the initial Linux endpoint agent

## Status
Accepted

## Context
The agent needs to be built and iterated on quickly by a solo developer,
while still doing real eBPF/audit-based collection.

## Decision
Implement the MVP agent in Python (using `bcc`/`libbpf`-python bindings
for eBPF, `python-audit`/log-tailing for auditd) as a systemd service.

## Alternatives Considered
- **Go agent:** better resource footprint, single static binary, common
  choice for production agents. Rejected for MVP because it would double
  the language surface for a solo developer already spending most of the
  budget on backend/AI work in Python.

## Consequences
Higher runtime overhead than a Go/C agent; acceptable at MVP demo scale
(single-digit endpoints). Flagged as a future rewrite candidate if
performance becomes a real constraint (see `MVP_SCOPE.md`).
