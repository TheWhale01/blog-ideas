---
title: Implement Proper Monitoring & Alerting
created: 2026-10-15
updated: 2026-11-07
type: todo
tags: [nixos, erebos, todo, improvement, infrastructure]
sources: ["live inspection 2026-11-07: alertmanager 0.31.1 + matrix-alertmanager active"]
---

# Implement Proper Monitoring & Alerting

- **Status**: In progress — discovered already 80% deployed (2026-11-07)
- **Priority**: Critical
- **Parent**: [[todo/index]]

## Reality check (was written assuming nothing existed)

`sys/services/monitoring/prometheus.nix` already contains a **complete alerting pipeline**, live and
healthy (`/-/healthy` = OK):

- `services.prometheus.alertmanager` — enabled, v0.31.1, port 9093, `openFirewall = false`
- routing → receiver `erebot` → `webhook_configs` → `services.matrix-alertmanager` (port 3004),
  homeserver `matrix.thewhale.fr`, `tokenFile = erebot.age`, `secretFile = matrix-alertmanager-webhook.age`,
  posting into a configured room ID
- Prometheus side: 2 alert rules already active (`ServiceCrashed` 1m/critical, `ServiceOffline` 3m/warning)

Synthetic end-to-end test fired 2026-11-07 (alert `WikiPipelineTest` injected via AM API, visible in
`/api/v2/alerts`) — **awaiting owner confirmation it rendered in the Matrix room**.

## Remaining work

- [ ] ⚠️ **Rotate the leaked webhook secret + remove it from git**: the query-string secret is
      committed in plaintext in a **public** repo (prometheus.nix `webhook_configs.url`). Anyone can
      forge alerts into the room. New secret must not return to the repo: keep it agenix-only and
      build the URL from a gitignored local file (e.g. `sys/secrets.local.nix`, mode 0600, imported
      optionally) — acceptable on-box (store is local-readable) vs global-public today
- [ ] Bind loopback (closes audit P2 remainder): `alertmanager.listenAddress = "127.0.0.1"` …
      live `ss` shows `*:9093`, `*:3004`, `*:3100` (Loki) still all-interfaces
- [ ] Expand rules beyond systemd state: disk >85 % (`/` btrfs + `/data`), sustained CPU, free mem,
      `node_xfs`/filesystem read-only remount, Prometheus target-missing; SMART rules later via
      [[ideas/Drive-Health-Monthly-Newsletter]]
- [ ] Loki LogQL error panels (Traefik 5xx, Nextcloud) — still to do
- [ ] Noise guard: daily `update-containers` restarts all `podman-*` — rules must ignore transient
      restarts (`for: 5m` or dependency-aware inhibit) — owner decision 2026-11-07, [[docs/erebos/containers]]
- [ ] Confirm Grafana path: current design is Prometheus-native rules → AM → Matrix; Grafana unified
      alerting is **optional**, don't run two alert brains

## Related

- [[docs/erebos/monitoring]] — the stack
- [[queries/security-audit]] — P2 binds + new secret-exposure finding
- [[ideas/Drive-Health-Monthly-Newsletter]] — future SMART alert source into this pipeline
