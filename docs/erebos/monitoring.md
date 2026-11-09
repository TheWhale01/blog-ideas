---
title: Monitoring
type: reference
created: 2026-10-15
updated: 2026-11-09
tags: [nixos, erebos, monitoring, observability]
sources: []
---

# Erebos — Monitoring Stack

The monitoring stack lives in `sys/services/monitoring/` and is composed by `sys/services/monitoring/default.nix`.

## Components

| File | Tool | Role |
|---|---|---|
| `prometheus.nix` | Prometheus | Time-series metrics and alerting rules |
| `grafana.nix` | Grafana | Dashboards, visualization, alerting UI |
| `loki.nix` | Loki 3.7.9 | Log aggregation + storage (filesystem chunks; **collector-independent**) |
| `alloy.nix` (pending — patch `erebos-alloy.patch`) | **Alloy** (promtail removed from nixpkgs — EOL, module assertion) | Log shipping. Repo file is still `promtail.nix`, `enable`d with **no config** → Loki zero series until the patch is applied. Final collector config: next section (validated with `nix eval` + `alloy validate`/`fmt`). |

## Log collection — Alloy config (patch delivered 2026-11-09, validated)

The `services.alloy` module (checked through nixpkgs master) has **no `settings` attrset** —
River is a DSL with no generator, unlike the attrset-driven `matrix-synapse`/`mas` configs.
The supported Nix-declarative pattern is **`environment.etc."alloy/config.alloy".text`**.
`services.alloy.configPath` exists but should stay at its default `/etc/alloy` (directory
mode): the module adds every `environment.etc."alloy/*.alloy"` file to
`systemd.services.alloy.reloadTriggers`, so `nixos-rebuild switch` **hot-reloads** Alloy.
Pointing `configPath` at a store path (e.g. `"${pkgs.writeText ...}"`) works but loses reload
(the module docs call this trade-off out explicitly). The module also grants the service the
`systemd-journal` supplementary group automatically, and **fails the whole initial load** if a
referenced file (e.g. the geoip mmdb) is missing — hence the crash-loop warnings below.

The final config lives in the pending patch **`/var/lib/hermes/workspace/erebos-alloy.patch`**
(`git am` it on a branch; replaces `promtail.nix` with `sys/services/monitoring/alloy.nix`).
It renders Alloy's River config from Nix the way `mas.nix` renders YAML — values interpolated
from module options rather than hardcoded:

- `loki.write "default"` push URL ← `config.services.loki.configuration` (currently
  `http://127.0.0.1:3100/loki/api/v1/push`)
- journal `host` label ← `config.networking.hostName`
- pipeline: `loki.source.journal "system"` (`path = "/var/log/journal"`, `max_age = "12h"`) →
  `discovery.relabel "journal"` promotes `__journal__systemd_unit` → `unit` →
  `loki.process "system"` (`stage.match` on `{unit="traefik.service"}`: `stage.json` extracts
  `status`/`request_path`/`client_ip`/`method`, `stage.labels` adds status/request_path/method) →
  `loki.write`. Geoip `stage.geoip` is included **commented out**: it opens its mmdb at config
  load and `/var/lib/geoip/GeoLite2-City.mmdb` does not exist today — uncommenting without the
  file bricks the service (crash loop, verified experimentally with `alloy run`).

Validation recipe (what the patch passed): `nix eval` of the rendered text from the actual
flake config, then `$A validate file.alloy` + `$A fmt` (exit 0 + silence), where
`A=/nix/store/*-grafana-alloy-*/bin/alloy`; component names must be unique across all
`/etc/alloy/*.alloy` files.

Notes:
- `format` is **not** a valid argument on `loki.source.journal` in Alloy 1.16 (a leftover from the
  promtail draft — `alloy validate` rejects it); journal lines flow as text and `stage.json` parses
  Traefik's messages fine.
- Re-validate any edit before applying: `$A validate file.alloy` where
  `A=/nix/store/*-grafana-alloy-*/bin/alloy` (exit 0 + silence = OK).
- Once this lands, the retention snippet above finally has something to prune; first load backfills
  only the last `max_age` (12 h) of journal — expect a burst on first start, quiet after.
- Queries to smoke-test: `{job="systemd-journal"} |= "Failed password"` (sshd), 
  `{unit="traefik.service", status=~"5.."}` (edge errors), `{host="erebos"} |~ "error"`.

## Loki retention — 1 month (paste into `loki.nix`, inside `services.loki.configuration`)

Loki 3.x needs **both** blocks — the period alone does nothing without an enabled compactor, and
the compactor refuses to start in 3.x without `delete_request_store`:

```nix
      compactor = {
        retention_enabled = true;
        delete_request_store = "filesystem";   # matches common.storage.filesystem
        working_directory = "${config.services.loki.dataDir}/compactor";
        compaction_interval = "10m";
      };
      limits_config = {
        retention_period = "720h";              # 1 month
        max_look_back_period = "720h";          # queries can't reach past the horizon either
      };
```

Verify after apply: `curl -s 127.0.0.1:3100/metrics | grep -m1 loki_build_info` (service alive) and
`curl -s 127.0.0.1:3100/metrics | grep loki_compactor` (compactor working). Existing chunks are
deleted lazily at the next compaction cycle once they age past 720 h — retention starts eating the
backlog only once logs actually flow, which needs the collector fix above first.

## Integration

The monitoring stack is wired into the NixOS config through:

- `sys/services/monitoring/default.nix` (imports all four modules);
- `sys/services/default.nix` (the monitoring module is imported here);
- Traefik dynamic config (Grafana UI exposed at `grafana.thewhale.fr`, behind Authentik).

## TODO

For current status and tasks, see [[todo/monitoring-alerting]].

## External Monitoring

Because Erebos is a server, it also relies on:

- **Netbird** (via nixos-modules) — VPN for safe remote management;
- **Traefik's** own `metrics.prometheus` endpoint (exposed on `127.0.0.1:8200`).

These are all part of the overall monitoring and observability picture.
