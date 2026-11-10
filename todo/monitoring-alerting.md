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
- **Priority**: Medium (core + security threads closed; remainder = soak, hygiene, optional alerts)
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
traefik JSON access logs parsed into `status`/`method`/`request_path` labels (+`client_ip` extracted);
**GeoIP LIVE** (`1966884`/`9b24091`): `stage.geoip` on traefik + sshd + fail2ban `Ban <IP>` streams,
`geoip_country_name` label + `geo_country`/`geo_city` structured metadata; DB auto-refreshed by
`geoipupdate.timer` (verified active, next run 2026-11-16). Prometheus scrape jobs: prometheus, node_exporter,
authentik, traefik, **alloy** (12345), **loki** (3100) — last two verified returning series.
ScrapeTargetDown therefore now auto-covers collector death. Grafana: Prometheus+Loki datasources
provisioned; terranix-managed dashboards = node_exporter (1860) + traefik (17346); **Journal Logs
dashboard now terraform-provisioned** (`3c69878`, after the v2→classic schema fix — owner confirmed live);
Grafana-managed dashboards-as-code = node_exporter + traefik (live catalog fetch) + geoip fork +
systemd-journal fork (`replaceStrings` DS binding);
terranix's 14055 entry removed.

## What's left

- [x] ~~Loki retention~~ — APPLIED `050df05` (720h; corrected snippet — `max_look_back_period`
      is gone in Loki 3.x, binary-verified).
- [~] **Loopback binds** — Alertmanager `9093` ✅ `11a4979`; Loki `3100`+gRPC ✅ `f13b255`
      (verified snippet). **Bridge `:3004`: owner declined** (2026-11-10) — firewall enabled, no
      internet forwarding; accepted residual: podman0/wt0 peers could reach :3004 (webhook is
      secret-gated; worst case alert noise/silence, not data). Snippet stays on file in
      [[docs/erebos/monitoring]] "Loopback binds" if posture changes. Jellyfin `:8096` → [[queries/security-audit]].
- [x] ~~Grafana UI cleanup~~ — done by owner manually 2026-11-10 (14055 + orphaned authentik).
- [x] ~~Alert-link suppression~~ — APPLIED `dc499f2` (link removed from alert message + room
      mention removed; verified by agent 2026-11-10).
- [x] ~~GeoIP enrichment~~ — LIVE (`1966884` + fail2ban branch `9b24091`); ban-map panel in fork;
      smoke test of first ban still worth doing (`fail2ban-client set sshd banip 8.8.8.8` → US tile →
      `unbanip`) — see [[docs/erebos/services/fail2ban]].
- [ ] **Noise guard soak** — after ~1 week, confirm daily `update-containers` restarts stay absorbed
      by `for: 5m` (no storm in the room); keep Grafana unified alerting OFF: single alert brain.
- [ ] **Issues room read-only** (bot-only writes) → [[todo/issues-room-readonly]]
- [x] ~~Drop `request_path` from traefik `stage.labels`~~ — DONE `98a56d4` (label never functioned:
      camelCase key mismatch — see [[docs/erebos/monitoring]] pitfall; query paths via `| json` instead).
- [x] ~~Pipeline-health alerts~~ — LIVE `98a56d4`: AlloyConfigNotLoaded, LokiDroppingEntries,
      LogsPipelineStalled (12 rules total, all healthy on live Prometheus).
- [x] ~~Whitelist additions~~ — LIVE `98a56d4` — but owner replaced the explicit podman names with
      `podman-.*`: watched units ≈30 → **46** (21 podman units now included, lidarr via `podman-lidarr`).
      **Soak must now watch daily `update-containers` against ALL podman units** — one >5 min container
      restart = a warning in the room.
- [x] ~~host label parity~~ — resolved by owner hardcoding `host = "erebos"` in the journal source;
      all journal streams share it (equivalent in effect).
- [ ] Pipeline-health alerts: `alloy_config_last_load_successful`, `loki_write_dropped_entries_total`
      rules (cheap insurance that silent log-pipeline death pages you).
- [ ] Post-apply render check on the geoip fork's duplicate gridPos/`id=7` (API persistence VERIFIED
      verbatim on throwaway 13.0.12; display-layer reflow unresolved) → [[docs/erebos/monitoring]] pitfall.
- [x] ~~drive_health group~~ LIVE 2026-11-10 ~22:21 (17 rules ok, smartctl scrape up=1).
      **First strike: sdb confirmed failing** (280 realloc/207 pending/190 uncorrectable, WD health-byte
      lies PASSED → sector rules caught it) — see [[ideas/drive-health-monthly-newsletter]] postmortem;
      NOTE: smart.nix/rules still unpushed (prod ahead of repo tip 98a56d4).
- [ ] fail2ban decisions → [[todo/fail2ban-tuning]]: aggressive-mode upgrade (needs home-IPv6/wt0
      ignoreIP FIRST — today's ignoreIP does NOT cover the login prefix `2a02:842a:3540:4201::…`),
      first web jail over traefik JSON lines.
- [ ] Later: SMART rules → [[ideas/drive-health-monthly-newsletter]]; `livekit`/`homepage-dashboard`/
      seerr/arr-stack are deliberately NOT in the ServiceOffline whitelist — add to the regex in
      `todo/alert-rules.yml` if you want them paged.

## Editing the rules — workflow

Source of truth: **`erebos:sys/services/monitoring/prometheus/config.nix`** (Nix attrsets, `watchedServices`
list drives the ServiceOffline regex; rendered by `pkgs.formats.yaml` into `ruleFiles` —
**promtool check now runs at nixos-rebuild time**). The attached `alert-rules.yml` is a generated
snapshot for reference. Tests: **`todo/alert-rules.test.yml`** (6 scenarios — re-verified SUCCESS
against the rendered 12-rule output 2026-11-10).

```bash
PT=$(ls /nix/store/*-prometheus-*-cli/bin/promtool | head -1)
$PT check rules todo/alert-rules.yml && $PT test rules todo/alert-rules.test.yml   # BEFORE deploy
curl -s 127.0.0.1:9090/api/v1/rules | grep -o '"health":"[^"]*"' | sort | uniq -c   # AFTER apply (expect 17x ok)
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
