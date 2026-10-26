---
title: Add Robust Backups
created: 2026-10-15
updated: 2026-10-22
type: todo
tags: [nixos, erebos, todo, improvement, infrastructure]
sources: []
---

# Add Robust Backups

- **Status**: Not started
- **Priority**: Critical
- **Parent**: [[todo/index]]

## Description
No backup solution is currently in place for the `/data` LVM volume, the service data directories, the databases, or the system/secrets configuration.

## Tasks
- [ ] Implement BorgBackup or Restic for `/data` and critical directories
- [ ] Configure remote storage (e.g., S3-compatible or secondary server)
- [ ] Schedule automated backups via `systemd-timer`
- [ ] Test restore procedures regularly (quarterly drill)
- [ ] Add weekly `borg check --verify-data` (alerting once [[todo/monitoring-alerting]] lands)

## Detailed notes
Scope must cover **both**:
- `/data` LVM volume (Nextcloud, Immich, Jellyfin, downloads), and
- service data dirs **outside** `/data` — notably `/var/lib/opengym/api` (see [[docs/erebos/services/opengym]]), `/var/lib/aurral`, etc.
- Exclude `/var/lib/containers` (images are re-pullable).

Databases: logical dumps, not file copies — `pg_dumpall` pre-hook covering `vaultwarden`, `nextcloud`, `authentik`, `matrix-synapse`, `mas`. Also back up the agenix-encrypted secrets repo and the flake repo itself.

Tooling: prefer BorgBackup (dedup + mature `services.borgbackup` NixOS module). Retention: 7 daily / 4 weekly / 12 monthly. Remote target: `erebos-stage` or S3-compatible (Hetzner/Scaleway/Backblaze). Store the repo key in agenix.

Acceptance: quarterly restore drill — full restore of one Postgres DB + one Borg mount verification — logged in this page's history.

## Related
- [[docs/erebos/index]]
- [[docs/erebos/services/opengym]]
- [[docs/erebos/secrets]]
