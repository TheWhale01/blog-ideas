---
title: Backups — specified scope
created: 2026-11-07
updated: 2026-11-07
type: todo
tags: [erebos, todo, improvement, infrastructure]
sources: ["owner spec 2026-11-07"]
---

# Backups — specified scope

- **Status**: Not started
- **Priority**: Critical
- **Parent**: [[todo/index]]

## Owner-confirmed facts (2026-11-07)

- `/data` is **NOT redundant** — the `whale--nas-data` name is legacy from a previous, separate NAS
  build; it is a plain LVM volume on this server (VG rename never done). A disk death loses `/data`.
- **Sensitive services needing backup: only `vaultwarden`, `immich`, `nextcloud`.** Everything else
  (Jellyfin, *arr suite, Transmission, Synapse/MAS, Grafana, openGym, slskd, n8n, Immich-alike extras)
  is explicitly out of scope.

## Scope

What MUST be in any backup set (for the three in-scope services):
1. **PostgreSQL logical dumps** of the DBs `vaultwarden`, `immich`, `nextcloud` (file copies of the
   pgdata dir are forbidden — use `pg_dump` pre-hooks in Borg/Restic `services.borgbackup` jobs)
2. **`/data/Immich`** (photo/video originals + thumbs)
3. **`/data/nextcloud`** (user files, data directory)
4. `/var/lib/vaultwarden` if it holds non-DB state (push/sends config; the credentials live in PG)
5. The agenix identity keys → covered separately by [[todo/age-keys-to-strategos]] (different mechanism)

## Design decisions left open

- [ ] Tool: BorgBackup (dedup, NixOS `services.borgbackup` module) vs Restic — lean Borg
- [ ] **Remote target: strategos** (Raspberry Pi, same LAN — [repo](https://github.com/TheWhale01/strategos), aarch64, nixpkgs 25.11 stable; sshd already up with authorized keys, receives borg over SSH) — it is
      off-box but not off-site: acceptable for accidental-deletion/first-order disk failure; document
      that a total-fire loss of both boxes is uncovered by design
- [ ] Retention: start 7 daily / 4 weekly / 12 monthly
- [ ] Exclude lists: `/data/downloads`, media trees not owned by immich/nextcloud, `/var/lib/containers`
- [ ] Restore drill: quarterly `borg extract` of one DB + one asset on the Pi, then verify login

## Related

- [[docs/erebos/architecture]] — §5 Backup posture (facts mirrored there)
- [[todo/age-keys-to-strategos]] — the companion key-material backup
- [[todo/monitoring-alerting]] — should alert on failed backup jobs once it exists
