---
title: Implement Proper Monitoring & Alerting
created: 2026-10-15
updated: 2026-11-07
type: todo
tags: [nixos, erebos, todo, improvement, infrastructure]
sources: ["live inspection + end-to-end owner confirmation 2026-11-07 (erebos 790c16a, 85c5ebe)"]
---

# Implement Proper Monitoring & Alerting

- **Status**: Core **complete & owner-confirmed** (2026-11-07, "all tests worked perfectly") —
  this note now tracks only the remainder.
- **Priority**: High (what's left is hardening + polish; the bind item below is the last open
  security thread → [[queries/security-audit]])
- **Parent**: [[todo/index]]

## Current state (live)

Prometheus rules → Alertmanager 0.31.1 → matrix-alertmanager 0.9.0 bridge → ops room as **@erebot**
(appservice bot). **9 rules / 3 groups**: `systemd_services` (crash catch-all + 25-unit ServiceOffline
whitelist), `host_resources` (disk 10 %/5 %, read-only-fs canary, memory, CPU, reboot),
`monitoring_self` (ScrapeTargetDown). Promtool unit-tested before deploy, firing correctly,
`send_resolved` on. Webhook secret is **agenix-only** (envsubst placeholder, rotated in `85c5ebe`).
Delivery verified twice: server-side (journal event IDs) and client-side (owner's eyes).

## What's left

- [ ] **Loopback binds** — Alertmanager `:9093`, bridge `:3004`, Loki `:3100` still listen on `*`
      (live `ss` 2026-11-07). `podman0` is a trusted network ⇒ any container breach could currently
      silence Alertmanager or read Loki. One `listenAddress = "127.0.0.1"` per service in
      `sys/services/monitoring/*`. (Jellyfin `:8096` tracked in [[queries/security-audit]].)
- [ ] **Alert-link suppression** — one-line `ALERT_LINKS` env entry, not yet applied (unit env
      verified unset). Snippet: [[docs/erebos/services]] "Maintenance recipes & Matrix gotchas".
- [ ] **Loki LogQL error panels** (Traefik 5xx, Nextcloud) on the Grafana side.
- [ ] **Noise guard soak** — after ~1 week, confirm daily `update-containers` restarts stay absorbed
      by `for: 5m` (no storm in the room); keep Grafana unified alerting OFF: single alert brain.
- [ ] **Issues room read-only** (bot-only writes) → [[todo/issues-room-readonly]]
- [ ] Later: SMART rules → [[ideas/Drive-Health-Monthly-Newsletter]]; `livekit`/`homepage-dashboard`/
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
- [[ideas/Drive-Health-Monthly-Newsletter]] — future SMART source into this pipeline
