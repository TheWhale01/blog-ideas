---
title: Implement Proper Monitoring & Alerting
created: 2026-10-15
updated: 2026-11-10
type: todo
tags: [nixos, erebos, todo, improvement, infrastructure]
sources: ["live inspection + end-to-end owner confirmation 2026-11-07 (erebos 790c16a, 85c5ebe); log-pipeline go-live verified on-host 2026-11-09/10 (erebos abc3f49..e0f8d57)"]
---

# Implement Proper Monitoring & Alerting

- **Status**: Core **complete & owner-confirmed** (2026-11-07, "all tests worked perfectly") —
  this note now tracks only the remainder.
- **Priority**: High (what's left is hardening + polish; the bind item below is the last open
  security thread → [[queries/security-audit]])
- **Parent**: [[todo/index]]

## Current state (live)

Prometheus rules → Alertmanager 0.31.1 (**loopback-bound since `11a4979`** ✅) → matrix-alertmanager
0.9.0 bridge → ops room as **@erebot** (appservice bot). **9 rules / 3 groups**: `systemd_services`
(crash catch-all + 25-unit ServiceOffline whitelist — includes `alloy`), `host_resources`
(disk 10 %/5 %, read-only-fs canary, memory, CPU, reboot), `monitoring_self` (ScrapeTargetDown).
Promtool unit-tested before deploy, firing correctly, `send_resolved` on. Webhook secret is
**agenix-only** (envsubst placeholder, rotated in `85c5ebe`). Delivery verified twice.

**Log pipeline now LIVE (was dark since promtail's nixpkgs removal)** — verified on-host 2026-11-09:
journald → **Alloy** (store-path config via `pkgs.writeText`, `configPath`, `e0f8d57` lineage) →
Loki :3100 — 217k+ lines received; streams: `{job="systemd-journal", host="erebos", unit=…}`;
traefik JSON access logs parsed into `status`/`method`/`request_path` labels (+`client_ip` extracted;
geoip stage parked — no mmdb on host). Prometheus scrape jobs: prometheus, node_exporter,
authentik, traefik, **alloy** (12345), **loki** (3100) — last two verified returning series.
ScrapeTargetDown therefore now auto-covers collector death. Grafana: Prometheus+Loki datasources
provisioned; terranix-managed dashboards = node_exporter (1860) + traefik (17346); **Journal Logs
dashboard = repo JSON (`sys/services/monitoring/dashboards/systemd-journal.json`, Grafana 13
kind/spec) manually imported — owner declined terraform/Nix auto-import for dashboards**;
terranix's 14055 entry removed.

## What's left

- [ ] **Loki retention NOT applied** — 1-month snippet (compactor + limits_config, both required on
      3.x) is in [[docs/erebos/monitoring]] but absent from `loki.nix`: disk growth is unbounded.
- [ ] **Loopback binds (remainder)** — Alertmanager + Loki retention done. Verified findings
      2026-11-10: `trustedInterfaces` (podman0, **wt0 = Netbird peers!**) accept ALL host ports at
      the top of the nft input chain ⇒ firewall rules can't fix; bind-level only. Ready-to-apply
      snippets (binary-verified): [[docs/erebos/monitoring]] "Loopback binds" — loki
      `http/grpc_listen_address = 127.0.0.1`; bridge via overrideAttrs APP_ADDRESS patch
      (`node --check`-verified artifact). Jellyfin `:8096` → [[queries/security-audit]].
- [ ] **Grafana UI cleanup** — 14055 (if it applied before removal) and orphaned authentik 14837
      (terranix resource was renamed to traefik in `11a4979`; the dashboard itself is now unmanaged)
      → delete in UI or re-add a managed entry deliberately.
- [ ] **Alert-link suppression** — one-line `ALERT_LINKS` env entry, not yet applied (unit env
      verified unset). Snippet: [[docs/erebos/services]] "Maintenance recipes & Matrix gotchas".
- [ ] **Noise guard soak** — after ~1 week, confirm daily `update-containers` restarts stay absorbed
      by `for: 5m` (no storm in the room); keep Grafana unified alerting OFF: single alert brain.
- [ ] **Issues room read-only** (bot-only writes) → [[todo/issues-room-readonly]]
- [ ] Optional polish: geoip (needs MaxMind-keyed mmdb → uncomment stage), pipeline-health alerts
      (`alloy_config_last_load_successful`, `loki_write_dropped_entries_total` rules), drop
      `request_path` from traefik `stage.labels` (high-cardinality vs Loki's index model),
      `host = "${config.networking.hostName}"` for stage parity.
- [ ] Later: SMART rules → [[ideas/drive-health-monthly-newsletter]]; `livekit`/`homepage-dashboard`/
      seerr/arr-stack are deliberately NOT in the ServiceOffline whitelist — add to the regex in
      `todo/alert-rules.yml` if you want them paged.

## Editing the rules — workflow

Source of truth: attached **`todo/alert-rules.yml`** (embeds verbatim into
`services.prometheus.rules = [ ''…'' ];` in `prometheus.nix`). Tests: attached
**`todo/alert-rules.test.yml`** (6 promtool scenarios — currently all green).

```bash
PT=$(ls /nix/store/*-prometheus-*-cli/bin/promtool | head -1)
$PT check rules todo/alert-rules.yml && $PT test rules todo/alert-rules.test.yml   # BEFORE deploy
curl -s 127.0.0.1:9090/api/v1/rules | grep -o '"health":"[^"]*"'                   # AFTER apply
```

Guard logic — do not regress:
- exporter `type` label = systemd **StartType** ⇒ keep `type!~"oneshot.*"` + `.service` name
  suffixes (this is what killed the target/oneshot/socket spam);
- offline rule stays **whitelist-based** — unfiltered `state="active" == 0` fires on legitimately
  stopped units; `llama-cpp` intentionally unwatched;
- `/data` sits ~13 % free ⇒ disk thresholds must stay **10 %/5 %** or the warning is on day one.

## Re-testing delivery (safe drills)

```bash
# format + delivery smoke, no for:-timer wait (~40 s to room):
NOW=$(date -u +%Y-%m-%dT%H:%M:%S.000Z)
curl -X POST http://127.0.0.1:9093/api/v2/alerts -H 'Content-Type: application/json' -d \
'[{"labels":{"alertname":"ServiceOffline","severity":"warning","name":"matrix-synapse.service",
"type":"notify","state":"active","instance":"127.0.0.1:9100","job":"node_exporter"},
"annotations":{"summary":"DRILL — injected, not real"},"startAt":"'"$NOW"'"}]'

# real trigger incl. resolved message (~7 min, zero user impact):
sudo systemctl stop prometheus-node-exporter; sleep 420; sudo systemctl start prometheus-node-exporter
```

Trouble map: journal shows send-200s but room empty → **v12 room-identity check** (Element →
Settings → Advanced vs `matrixRooms`); `M_FORBIDDEN`/`M_UNKNOWN_TOKEN` in the bridge journal →
appservice invite + token re-mint recipes in [[docs/erebos/services]] "Maintenance recipes &
Matrix gotchas".

## History

Full 2026-11-07 debugging story (HTTP 500s → appservice ghost account → room membership → v12 link
confusion → secret leak & rotation) lives in **this file's git history** + the 2026-11-07 `log.md`
entries. Durable knowledge was folded into [[docs/erebos/services]] and [[docs/erebos/secrets]].
Slimmed 2026-11-07 at owner request: the page shows the remainder at a glance.

## Related

- [[docs/erebos/services]] — pipeline description, maintenance recipes, gotchas
- [[queries/security-audit]] — P2 binds (open); webhook-leak finding (resolved)
- [[ideas/drive-health-monthly-newsletter]] — future SMART source into this pipeline
