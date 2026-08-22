# Endpoint Agent Specification (Linux, MVP)

## Lifecycle

```text
Install → Enroll (one-time token → agent JWT) → Heartbeat loop →
Collect → Normalize → Local Buffer → Batch Ship → Retry on failure →
Apply config updates (from heartbeat response) → Uninstall (revoke credential)
```

- **Install:** systemd service (`aegisx-agent.service`), runs as a
  dedicated low-privilege user plus the specific capabilities needed
  (`CAP_BPF`, `CAP_PERFMON`/`CAP_SYS_ADMIN` depending on kernel version,
  `CAP_AUDIT_READ`) rather than root.
- **Enrollment:** operator generates a short-lived, single-use enrollment
  token in the dashboard; agent exchanges it once for a long-lived signed
  agent credential (JWT, rotated periodically). The enrollment token itself
  is never reused and is invalidated after first use or expiry.
- **Heartbeat:** every 30s, agent reports liveness + version + health;
  server response can carry configuration updates (collection toggles,
  sampling rate, rule/allowlist updates) — pull-based, not agent-initiated
  remote code execution.
- **Local buffering:** events queue to an on-disk SQLite ring buffer sized
  to survive a configurable outage window (default 24h); oldest events are
  dropped (and counted) once the buffer is full rather than blocking the host.
- **Shipping:** batched, gzip-compressed HTTPS POST every 5s or when batch
  size threshold is hit, whichever first. Exponential backoff with jitter
  on failure (max interval capped).

## Data Collected (data-minimized)

| Category | Fields |
|---|---|
| System | hostname, OS/kernel version, CPU/mem/disk summary, running services, health |
| Process | pid, ppid, executable path, command line (redact known secret-bearing flags), user, start/end timestamp, exit code |
| Network | src/dst IP, ports, protocol, DNS query name (not full payload) |
| Authentication | user, result, source, timestamp (no passwords, ever) |
| Filesystem | path, action, hash of changed file (not full file content), acting process |
| Low-level | eBPF/audit event type, associated syscall, kernel timestamp |

Explicitly **not** collected: file contents, keystrokes, clipboard,
screenshots, browser history, personal documents. Command-line arguments
matching known secret patterns (`--password`, `--token`, etc.) are redacted
client-side before leaving the host.

## Collection Sources

| Source | Used for | Fallback if unavailable |
|---|---|---|
| eBPF (execve, connect, open/unlink probes) | process, network, file events | auditd rules |
| auditd | process, auth, privileged syscalls | procfs polling (reduced fidelity) |
| procfs | process/system inventory, health | — |
| inotify/fanotify | file create/modify/delete | periodic filesystem diff (last resort) |

Every event's `source` field records which mechanism produced it so
downstream detection can weight confidence accordingly — a procfs-sourced
"process start" is marked lower-fidelity than an eBPF-sourced one, and this
is stated in the UI rather than hidden.

## Security

- Agent credential stored with restrictive filesystem permissions
  (`0600`, owned by the service user), never logged.
- Transport is TLS; the ingestion endpoint's certificate is pinned/verified,
  not just "any HTTPS."
- Agent cannot receive or execute arbitrary commands from the server —
  only a fixed, versioned set of configuration fields are accepted from
  heartbeat responses, validated against a strict schema.
- Local buffer is not world-readable.
