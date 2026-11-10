---
title: Backups — Borg to strategos
created: 2026-11-07
updated: 2026-11-07
type: todo
tags: [erebos, strategos, todo, improvement, infrastructure, borg]
sources: ["owner spec 2026-11-07; tool decision borg 2026-11-07; paths verified from erebos repo 85c5ebe"]
---

# Backups — Borg to strategos

- **Status**: CORRECTED + FINAL SNIPPETS DELIVERED 2026-11-10 (validated option-by-option against
  pinned module `b40629e/nixos/modules/services/backup/borgbackup.nix`, 967 lines). Owner applies.
  The 2026-11-07 draft below was superseded — its option names (`environmentFile`, flat `prune.keep-daily`,
  `initializeAllRepositories`, `preCheckForCreate`) DO NOT EXIST in this module revision. See CHAT
  snippets / corrected config here:
- **Priority**: 🔴 Critical-elevated — sdb dying (see [[ideas/drive-health-monthly-newsletter]] postmortem)
  and **no backup exists yet**: first borg run = the rescue copy + damage scan.
- **Parent**: [[todo/index]]

## Corrections vs the 2026-11-07 draft (all module-verified 2026-11-10)

1. `environmentFile` → doesn't exist; secrets go via `environment` attrset:
   `BORG_PASSCOMMAND = "cat /run/agenix/borg-passphrase"` (passphrase never in store),
   `BORG_RSH = "ssh -i <agenix key path> -o UserKnownHostsFile=/etc/borgbackup/strategos-known_hosts"`.
2. Prune: `prune.keep = { daily = 7; weekly = 5; monthly = 12; };` (nested; mkKeepArgs renders `--keep-*`).
3. Init: `doInit` (default true) auto-`borg init --encryption repokey-blake2` on first run.
4. Path guards: use a `preHook` `test -d` (no `preCheckForCreate` option).
5. **strategos repos module prepends the restriction itself**: `authorizedKeys = [ "<bare pubkey>" ]` —
   do NOT hand-write `command="borg serve…"` prefixes (double-restriction breaks auth).
6. **Immich DB reality**: `services.immich` module injects `ensureDatabases = ["immich"]` into the
   HOST postgres (verified: no `immich-database.service` unit exists; `immich-server` + shared
   `postgresql.service` confirmed via journal). `pg_dump -Fc immich` on the host cluster is correct.
7. Nextcloud: container bind `/data/nextcloud:/var/www/html` confirmed; DB type = sqlite vs the
   vestigial pg `nextcloud` db — UNRESOLVED, preHook handles both branches; confirm with
   `podman exec -u www-data nextcloud php occ config:get database:type`.
8. **NEW HARD REQUIREMENT**: strategos has ONLY an SD-card root (cloud-image placeholder UUID) —
   the borg repo needs a dedicated USB disk; SD card is not a backup medium.
9. `appendFailedSuffix = true`: archives become official only after a fully-successful create —
   ideal on a dying drive (partial archives stay `.failed` instead of masquerading as backups).
10. Failure alerting is FREE: job unit reaching `state=failed` is caught by the existing generic
    `ServiceCrashed` rule — borg WARNINGS (exit 1) are ignored by the module by default; the
    journal still lists unreadable files = the damage map (`borgbackup-job-… | grep -i error`).

## Owner-confirmed facts (2026-11-07)

- `/data` is **NOT redundant** — the `whale--nas-data` name is legacy from a previous, separate NAS
  build; it is a plain LVM volume on this server. A disk death loses `/data`.
- **Sensitive services needing backup: only `vaultwarden`, `immich`, `nextcloud`.** Everything else
  (Jellyfin, *arr suite, Transmission, Synapse/MAS, Grafana, openGym, slskd, n8n) is out of scope.
- Remote target: **strategos** (Pi 3B+, same LAN, sshd up). Off-box, NOT off-site — deliberate;
  a total-fire loss of both boxes is uncovered by design.

## Scope — verified service reality (2026-11-07, from the flake)

| Service | Deployment | Database | Files to capture |
|---|---|---|---|
| vaultwarden | native | **PostgreSQL** db `vaultwarden` (socket) | `/var/lib/vaultwarden` (rsa keys, sends, attachments) |
| immich | native | **PostgreSQL** db `immich` | `/data/Immich` (originals, thumbs, uploads) |
| nextcloud | podman (`nextcloud` official image) | **SQLite inside the data dir** — `/data/nextcloud/data/nextcloud.db` (no external DB configured in `sys/containers/nextcloud.nix`) | `/data/nextcloud` (whole `/var/www/html`: config.php, apps, data) |

⚠️ Earlier draft wrongly assumed Nextcloud-on-Postgres — corrected. **Verify the sqlite path once**:
`ls -l /data/nextcloud/data/nextcloud.db`.
Never copy pgdata dirs — logical dumps only.

## Implementation — NixOS `services.borgbackup.jobs` (paste into e.g. `sys/services/backups.nix`)

Prereqs: 1) an ed25519 backup keypair; 2) agenix: `borg-backup.age` (private key) +
`borg-env.age` (KEY=VALUE: `BORG_PASSPHRASE=*** — add both to
`sys/secrets.nix` (owner-readable, 0400); 3) on **strategos**: enable SSH + the repo (below) + a
user `borg` with the **public** key in `authorized_keys`, restricted:
`command="borg serve --restrict --append-only",no-agent-forwarding,no-port-forwarding ssh-ed25519 AAAA…`

```nix
{ config, pkgs, vars, ... }:
let
  pg = config.services.postgresql.package;
in {
  age.secrets.borg-backup.file = ../secrets/shared/borg-backup.age;   # private key
  age.secrets.borg-env.file  = ../secrets/shared/borg-env.age;        # BORG_PASSPHRASE + BORG_RSH

  services.borgbackup.jobs."erebos-to-strategos" = {
    repo = "borg@strategos:/srv/borg/erebos";
    environmentFile = config.age.secrets.borg-env.path;  # BORG_RSH="ssh -i /run/agenix/borg-backup -o UserKnownHostsFile=…"
    paths = [
      "/var/lib/vaultwarden"
      "/data/Immich"
      "/data/nextcloud"
      "/run/borg-dumps"
    ];
    preHook = ''
      mkdir -p /run/borg-dumps
      runuser -u postgres -- ${pg}/bin/pg_dump -Fc vaultwarden -f /run/borg-dumps/vaultwarden.dump
      runuser -u postgres -- ${pg}/bin/pg_dump -Fc immich    -f /run/borg-dumps/immich.dump
      podman exec -u www-data nextcloud php occ maintenance:mode --on || true
      ${pkgs.sqlite}/bin/sqlite3 /data/nextcloud/data/nextcloud.db \
        ".backup '/run/borg-dumps/nextcloud.db'"
    '';
    postHook = ''
      podman exec -u www-data nextcloud php occ maintenance:mode --off || true
      rm -f /run/borg-dumps/* ; rmdir /run/borg-dumps || true
    '';
    exclude = [ "*.tmp" "/data/nextcloud/data/updater-*" ];
    compression = "zstd,10";
    startAt = "Sun 03:30";            # weekly; tmpfs /run dumps are ephemeral & consistent
    initializeAllRepositories = true; # creates the repo on first run (key = agenix'd, passphrase required)
    prune = { keep-daily = 7; keep-weekly = 5; keep-monthly = 12; };
    preCheckForCreate = "test -d /data/Immich -a -d /data/nextcloud -a -d /var/lib/vaultwarden";
  };

  systemd.services.borgbackup-job-erebos-to-strategos.after =
    [ "network-online.target" "postgresql.service" "podman-nextcloud.service" ];
}
```

strategos side (`configuration.nix`, works with nixpkgs 25.11):

```nix
services.borgbackup.repos."erebos" = {
  path = "/srv/borg/erebos";
  authorizedKeys = [ ''command="borg serve --restrict --append-only",no-agent-forwarding,no-x11-forwarding,no-user-rc ssh-ed25519 AAAA…(public) comment borg-erebos'' ];
};
```

Notes:
- Pi 3B+ constraint: borg **serve is fine** (transfer + keyed chunking only, no crypto-heavy prune);
  do prune client-side (this job config already does — client is Erebos). Expect slow first push of
  `/data/Immich` (dedup makes subsequent weeks cheap).
- Restore drill (quarterly, item from original spec): on the Pi
  `borg list repo`, client-side `borg extract --dry-run`; vaultwarden restore = `pg_restore` into the
  socket db; immich same; nextcloud = drop sqlite file back + `occ maintenance:mode` dance.
- Failure alerting: once wired, `onFailure = [ "matrix-alertmanager…" ]` no — better: a
  `ScrapeTargetDown`-style blackbox or a systemd unit-fail rule; hook into
  [[todo/monitoring-alerting]] "What's left" (failed-backup alert was always the plan).

## Can Borg push archives to a git repository?

**No — and don't try.** Borg maintains a deduplicated, encrypted **append-only archive repo**
(`borg serve` over SSH); it has no git transport. Pushing `.borg` archives into git(hub) would be
worst-of-both: every pack is large binary churn, git keeps all revisions forever (GitHub warns >1 GB
repos, blocks large files), and you'd leak the existence (not the contents — they're encrypted) of
backups into a VCS. Division of labour in this fleet: **git = configs/identity/docs (the flake,
this wiki), borg = user data**. For *off-site* coverage later, the answer is a second Borg repo
(Borgbase, or a Hetzner storage box with `borg-base` support), not git. Borg 2.x adds `borg sync`
(repo-to-repo) if you ever split targets.

## Related

- [[docs/erebos/architecture]] — backup posture (facts mirrored there)
- [[todo/age-keys-to-strategos]] — companion key-material backup (borg passphrase + key MUST be in it)
- [[todo/monitoring-alerting]] — alert on failed backup jobs once wired
- [[docs/strategos/services]] — add borg repo section when applied
