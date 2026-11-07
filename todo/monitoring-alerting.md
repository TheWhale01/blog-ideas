---
title: Implement Proper Monitoring & Alerting
created: 2026-10-15
updated: 2026-11-07
type: todo
tags: [nixos, erebos, todo, improvement, infrastructure]
sources: []
---

# Implement Proper Monitoring & Alerting

- **Status**: Not started
- **Priority**: Critical
- **Parent**: [[todo/index]]

## Description
The monitoring stack (Prometheus + Grafana + Loki + Promtail) is deployed on Erebos but lacks alerting rules and structured log analysis. Metrics are collected; nothing acts on them.

## Tasks
- [ ] Add `alertmanager` to the monitoring stack
- [ ] Define Prometheus alert rules (CPU, disk, memory, service health)
- [ ] Configure Loki for structured log queries (filter errors from Traefik/Nextcloud)
- [ ] Set up Grafana unified alerting with Matrix notifications (use existing `matrix-alertmanager-webhook` secret)
- [ ] Fire one synthetic alert end-to-end and confirm it lands in the Matrix room

## Detailed notes
Deploy `alertmanager` as a new module under `sys/services/monitoring/` and import it in `default.nix`.

Rules to start with:
- Disk >85% on `/` (btrfs) and `/data` (LVM)
- Sustained CPU >90% for 15m
- Free memory <256 MB
- `podman-*.service` units failed
- Traefik backend-down

Loki: build LogQL queries per service (Traefik 5xx spikes, Nextcloud login failures) and pin them as Grafana panels.

Alert routing: Grafana unified alerting → Alertmanager → Matrix via the `matrix-alertmanager-webhook` age secret already present in `secrets.nix`.

Caveat: the daily `update-containers` unit restarts every `podman-*` service — alert rules must treat transient podman restarts as noise (update pipeline stays as-is, owner decision 2026-11-07, [[docs/erebos/containers]]).

## Related
- [[docs/erebos/monitoring]]
- [[docs/erebos/containers]] — daily update-containers behavior
