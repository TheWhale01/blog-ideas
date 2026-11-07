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

- [x] ~~Fire one synthetic alert end-to-end~~ attempted 2026-11-07 — **pipeline is BROKEN at the
      bridge→Matrix leg**: direct bridge POST with the correct webhook envelope (`receiver: erebot`)
      returns `{"result":"error"}` + HTTP 500, i.e. the room for `erebot` IS matched but the Matrix
      send fails. Prime suspects: (a) `@erebot` not joined to room `!FVqWRrn1hvNO8Ykzs3xEdSP00B3mokJ7Ad38LkqNL0Y`
      (note: configured room ≠ the Odin DM room — alerts go to a separate room), (b) dead/expired
      access token in `erebot.age`. Diagnose: `sudo journalctl -u matrix-alertmanager -n 50`
      (expect `M_UNKNOWN_TOKEN` / 403), then re-login + `agenix -e` the token file if needed.
      The webhook **secret** itself is confirmed working (bridge accepted our POST without 401).
- [ ] ⚠️ **Fix rule spam before enabling delivery**: live AM currently holds firing `ServiceOffline`
      alerts for `systemd-update-done.service` (oneshot), `poweroff.target` (target) and
      `virtlxcd-admin.socket` (socket) — the `unit-include` regex in node_exporter extraFlags is not
      filtering as intended, and the rule expression ignores unit type. Fix:
      `expr: node_systemd_unit_state{name=~".+\\.service", type!~"oneshot|oneshot-reload", state="active"} == 0`
      (and audit why `--collector.systemd.unit-include` isn't effective in the running node_exporter)
- [ ] Rotate the leaked webhook secret + remove it from git: currently committed plaintext in the
      **public** repo (`prometheus.nix` receiver URL) and *working* — anyone can forge alerts. New
      secret lives agenix-only; build the URL from a gitignored local file
      (`sys/secrets.local.nix`, 0600, optional import)
- [ ] Bind loopback (closes audit P2 remainder): AM 9093, bridge 3004, Loki 3100 — live `ss` still `*`
- [ ] Expand rules beyond systemd state
- [ ] Loki LogQL error panels
- [ ] Noise guard + confirm single alert brain (Grafana unified alerting stays optional)

## Related

- [[docs/erebos/monitoring]] — the stack
- [[queries/security-audit]] — P2 binds + new secret-exposure finding
- [[ideas/Drive-Health-Monthly-Newsletter]] — future SMART alert source into this pipeline
