# LockDev observability agent guide

This repository owns the observability application stack on Proxmox LXC 900 (`observability`, `192.168.1.40`). Read the [cross-service agent context](https://github.com/MikeLockz/lockdev-docs/blob/trunk/docs/tech/agent-context.md) (local checkout: `~/Development/lockdev-docs/docs/tech/agent-context.md`) for the host map and dependencies.

Proxmox guest, backup, and storage changes belong to `lockdev-infra`; DNS changes belong to `lockdev-infra/cloudflare/`. Inspect live state and `git status` before editing. Validate affected Compose services and telemetry routes after a scoped change; never commit credentials.
