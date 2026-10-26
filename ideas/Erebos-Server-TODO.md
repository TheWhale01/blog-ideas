---
title: Erebos Server TODO List
created: 2026-10-15
updated: 2026-10-22
type: idea
tags: [nixos, erebos, homelab, todo, improvement]
sources: []
---

# Erebos Server TODO List

High-priority improvements for the Erebos NixOS server, ordered by impact.

## 1. Implement Proper Monitoring & Alerting
- **Status**: Not started
- **Priority**: Critical
- **Description**: The monitoring stack (Prometheus + Grafana + Loki + Promtail) is deployed but lacks alerting rules and structured log analysis.
- **Tasks**:
  - [ ] Add `alertmanager` to the monitoring stack
  - [ ] Define Prometheus alert rules (CPU, disk, memory, service health)
  - [ ] Configure Loki for structured log queries (filter errors from Traefik/Nextcloud)
  - [ ] Set up Grafana unified alerting with Matrix notifications (use existing `matrix-alertmanager-webhook` secret)
- **Detailed notes**: Deploy `alertmanager` as a new module under `sys/services/monitoring/` and import it in `default.nix`. Rules to start with: disk >85% on `/` (btrfs) and `/data` (LVM), sustained CPU >90% for 15m, free memory <256 MB, `podman-*.service` units failed, and Traefik backend-down. Loki: build LogQL queries per service (Traefik 5xx spikes, Nextcloud login failures) and pin them as Grafana panels. Alert routing: Grafana unified alerting → Alertmanager → Matrix via the `matrix-alertmanager-webhook` age secret already present in `secrets.nix`. Acceptance test: fire one synthetic alert end-to-end and see it land in the Matrix room. Beware restart-chains — the alerting work should land after item 3 (image updates) or be scoped to ignore transient podman restarts.
- **Related**: [[docs/erebos/monitoring]]

## 2. Add Robust Backups
- **Status**: Not started
- **Priority**: Critical
- **Description**: No backup solution is currently in place for the `/data` LVM volume or system configuration.
- **Tasks**:
  - [ ] Implement BorgBackup or Restic for `/data` and critical directories
  - [ ] Configure remote storage (e.g., S3-compatible or secondary server)
  - [ ] Schedule automated backups via `systemd-timer`
  - [ ] Test restore procedures regularly
- **Detailed notes**: Scope must cover **both** `/data` (LVM volume: Nextcloud, Immich, Jellyfin, downloads) **and service data dirs outside it** — notably `/var/lib/opengym/api` (openGym stores its data there, see [[docs/erebos/services/opengym]]), `/var/lib/containers` excluded (images are re-pullable), Postgres dumps for `vaultwarden`, `nextcloud`, `authentik`, `matrix-synapse`, `mas` (a `pg_dumpall` pre-hook, not just file copy), agenix-encrypted secrets repo, and the flake repo itself. Prefer BorgBackup (dedup + `services.borgbackup` NixOS module is mature). Retention: 7 daily / 4 weekly / 12 monthly. Remote target: `erebos-stage` or S3-compatible (e.g. Hetzner/Scaleway/Backblaze). Store the repo key in agenix. Quarterly restore drill: full restore of one Postgres DB + one Borg mount verification, logged here in this page. Acceptance: an automated weekly `borg check --verify-data` run that alerts via item 1 once alerting exists.
- **Related**: [[docs/erebos/index]], [[docs/erebos/services/opengym]]

## 3. Automate Podman Image Updates
- **Status**: Partial (basic `update-containers` service exists)
- **Priority**: High
- **Description**: The current `podman-auto-update` service lacks robustness and notifications.
- **Tasks**:
  - [ ] Replace with official `podman-auto-update` or custom script
  - [ ] Pull images only if changed (avoid unnecessary restarts)
  - [ ] Validate image signatures where available
  - [ ] Add failure notifications via Matrix
- **Detailed notes**: The current daily oneshot in `sys/containers/default.nix` pulls *every* image and restarts all `podman-*` units regardless of change. Move to Podman-native auto-update: label containers with `io.containers.autoupdate=image` (or `registry` for digest pinning) and run `podman auto-update --dry-run` first to verify only changed images trigger restarts. For systemd-managed podman units, use `--type=systemd`. Signatures: cosig/sigstore support is patchy across the GHCR images in use — treat as best-effort (validate for images that publish them, e.g. linuxserver.io). Failure path: on `podman auto-update` error, keep the old container running (no loop), emit a Matrix notification (webhook secret already exists). Add a weekly "newest digests" report to the same channel so updates are observable, not silent. Depends on nothing else; can land before item 1.
- **Related**: [[docs/erebos/containers]]

## 4. Centralize and Secure Secrets Management
- **Status**: Partial (agenix in use, but manual)
- **Priority**: High
- **Description**: Secrets are split across directories with manual rotation and access control.
- **Tasks**:
  - [ ] Implement regular secret rotation (quarterly)
  - [ ] Use `age` key groups to restrict decryption access
  - [ ] Backup age keys securely (encrypted, offline storage)
  - [ ] Document secret ownership and access policies
- **Detailed notes**: Rotation cadence: quarterly for service credentials exposed to the internet ( Authentik, Nextcloud, Vaultwarden DB creds, SMTP tokens), on any offboarding/suspicion otherwise; record last-rotation date per secret. Recipients: today most secrets list `hades` + services broadly — split identity keys so each container/systemd unit only decrypts its own secret (agenix `age.secrets.<name>.recipientsFile` per service), shrinking blast radius. Key backup: master age identity (ssh-key-derived `age1...`) + per-service keys exported, encrypted to a second offline key, stored outside Erebos (password manager + printed/passphrase-protected file) — losing it makes every `.age` file unrecoverable. Document ownership by extending the existing table in [[docs/erebos/secrets]] with columns: owner, recipients, last rotated, rotation cadence. Note: the terraform state (item 5) becomes a secret-adjacent surface once remote — lock down its bucket/DB access the same way.
- **Related**: [[docs/erebos/secrets]]

## 5. Improve Terraform/OpenTofu Workflow
- **Status**: Partial (terranix + OpenTofu in use)
- **Priority**: Medium
- **Description**: IaC workflow lacks drift detection and state management.
- **Tasks**:
  - [ ] Add `tofu plan` check in CI (GitHub Actions) for drift detection
  - [ ] Store OpenTofu state in remote backend (S3/PostgreSQL)
  - [ ] Add `nix run .#plan` app for previewing changes
  - [ ] Document state locking and recovery procedures
- **Detailed notes**: State first: current `terraform.tfstate` lives locally on Erebos — single point of failure and no locking. Move to PostgreSQL backend (already runs Postgres for services) or S3-compatible with locking; do it with `tofu init -migrate-state`, validate `tofu plan` shows zero changes after migration, keep a pre-migration state copy. CI: GitHub Actions job running `nix flake check` + terranix eval + `tofu plan -detailed-exitcode` (exit 2 = drift → fail the job / open a comment), using a read-only provider token stored as repo secret. Flake apps: mirror existing `apply`/`apply-stage` with `plan`/`plan-stage` apps in `flake.nix` so previewing is one command and applies stay deliberate. Recovery doc: how to re-import state (backend re-init, `tofu import` for drifted Authentik resources, worst case rebuild via terraform modules since everything is declared). Ordering note: needs item 4's recipient discipline because provider tokens live in `secrets/{env}/terraform/`.
- **Related**: [[docs/erebos/terraform]]

## 6. Housekeeping: fix `WIKI_PATH` for the blog-ideas wiki
- **Status**: Not started
- **Priority**: Low
- **Description**: The `WIKI_PATH` environment variable still points to `/home/hades/code/blog-ideas`, which the `hermes` user cannot access. The working copy of the wiki (Obsidian vault + git repo) is `/var/lib/hermes/workspace/blog-ideas/`. Until this is fixed, any tooling that resolves `$WIKI_PATH` (llm-wiki skill defaults, cron jobs) would target the inaccessible path.
- **Tasks**:
  - [ ] Set `WIKI_PATH=/var/lib/hermes/workspace/blog-ideas` in `/var/lib/hermes/.hermes/.env` (read by the llm-wiki skill)
  - [ ] Clear/override the stale export found in shell session environments
  - [ ] Verify with a new session that `echo $WIKI_PATH` resolves the workspace copy
- **Detailed notes**: The stale value was found only in the agent's persisted terminal-session snapshot environment — not in `~/.hermes/.env`, `config.yaml`, or the gateway process environ — so it likely leaked from a shell export made during a past session. Fixing `.env` makes the correct value authoritative for the llm-wiki skill; the session snapshot can be reset by starting a fresh terminal session (`/new` or dropping the scratch snapshot). If Obsidian on the desktop side should keep editing `/home/hades/code/blog-ideas` as its vault, that copy and the workspace copy must converge on the same git remote (or use Obsidian Sync) — otherwise edits will diverge and pushes from the agent will conflict. Verify: `hermes` terminal in a new session → `echo $WIKI_PATH` → `/var/lib/hermes/workspace/blog-ideas`; lint pass should find pages via the env var.
- **Related**: [[index]] — the catalog that llm-wiki tooling maintains via `$WIKI_PATH`

---

*Last updated: 2026-10-22*
