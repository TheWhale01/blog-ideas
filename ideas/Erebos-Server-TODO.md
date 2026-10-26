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
- **Related**: [[docs/erebos/index]]

## 3. Automate Podman Image Updates
- **Status**: Partial (basic `update-containers` service exists)
- **Priority**: High
- **Description**: The current `podman-auto-update` service lacks robustness and notifications.
- **Tasks**:
  - [ ] Replace with official `podman-auto-update` or custom script
  - [ ] Pull images only if changed (avoid unnecessary restarts)
  - [ ] Validate image signatures where available
  - [ ] Add failure notifications via Matrix
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
- **Related**: [[docs/erebos/terraform]]

## 6. Housekeeping: fix `WIKI_PATH` for the blog-ideas wiki
- **Status**: Not started
- **Priority**: Low
- **Description**: The `WIKI_PATH` environment variable still points to `/home/hades/code/blog-ideas`, which the `hermes` user cannot access. The working copy of the wiki (Obsidian vault + git repo) is `/var/lib/hermes/workspace/blog-ideas/`. Until this is fixed, any tooling that resolves `$WIKI_PATH` (llm-wiki skill defaults, cron jobs) would target the inaccessible path.
- **Tasks**:
  - [ ] Set `WIKI_PATH=/var/lib/hermes/workspace/blog-ideas` in `/var/lib/hermes/.hermes/.env` (read by the llm-wiki skill)
  - [ ] Clear/override the stale export found in shell session environments
  - [ ] Verify with a new session that `echo $WIKI_PATH` resolves the workspace copy

---

*Last updated: 2026-10-22*
