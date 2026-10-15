---
title: Erebos Server TODO List
created: 2026-10-15
updated: 2026-10-15
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

---

*Last updated: 2026-10-15*
