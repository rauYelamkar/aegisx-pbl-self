# ADR-0004: eBPF + Linux Audit, no custom kernel module

## Status
Accepted

## Context
Layer 1 (kernel/low-level telemetry) is a core differentiator, but writing
and maintaining a custom kernel module is a significant risk/maintenance
burden disproportionate to a solo college MVP.

## Decision
Use eBPF programs attached to existing, verified hook points (execve,
connect, file open/unlink) plus `auditd` rules for process, network, file,
and privileged-syscall telemetry. Fall back to procfs polling + inotify
where eBPF is unavailable, with the degraded source explicitly recorded
per event.

## Alternatives Considered
- **Custom kernel module (LKM):** would give the deepest visibility but
  introduces kernel-crash risk, requires per-kernel-version maintenance,
  and is explicitly discouraged by the project's own constraints ("Do not
  unnecessarily create custom kernel modules").

## Consequences
AEGISX cannot claim complete kernel protection — this is documented
explicitly in `ARCHITECTURE.md` and `SECURITY.md` rather than overclaimed.
