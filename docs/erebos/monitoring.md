---
title: Monitoring
type: reference
created: 2026-10-15
updated: 2026-11-07
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
| `promtail.nix` | **Alloy** (promtail removed from nixpkgs — EOL, module assertion) | Log shipping. Until 2026-11-09 the file was `enable` with **no config** → Loki zero series. Working collector config below (validated with `alloy validate` against 1.16.0). |

## Log collection — Alloy config (validated 2026-11-09, Alloy 1.16.0)

The module reads config from **`environment.etc."alloy/config.alloy"`** (NOT a `settings` option);
every `.alloy` file in that dir is a reload-trigger, so switches hot-reload Alloy. The module also
grants the service the `systemd-journal` supplementary group automatically — no manual group work.
Replace the contents of `sys/services/monitoring/promtail.nix` with:

```nix
{ ... }:
{
  services.alloy.enable = true;

  environment.etc."alloy/config.alloy".text = ''
    discovery.relabel "journal" {
      targets = []

      rule {
        source_labels = ["__journal__systemd_unit"]
        target_label  = "unit"
      }
    }

    loki.source.journal "system" {
      path          = "/var/log/journal"
      max_age       = "12h"
      labels        = { job = "systemd-journal", host = "erebos" }
      relabel_rules = discovery.relabel.journal.rules
      forward_to    = [loki.process.system.receiver]
    }

    loki.process "system" {
      forward_to = [loki.write.default.receiver]

      // Traefik JSON access logs -> structured labels (accessLog.format=json is set in traefik.nix)
      stage.match {
        selector = "{unit=\\"traefik.service\\"}"

        stage.json {
          expressions = {
            status       = "DownstreamStatus",
            request_path = "request_path",
            client_ip    = "ClientHost",
            method       = "RequestMethod",
          }
        }

        stage.labels {
          values = {
            status       = "",
            request_path = "",
            method       = "",
          }
        }

        // Optional geoip — requires /var/lib/geoip/GeoLite2-City.mmdb, which does NOT exist today
        // (the old promtail draft referenced it anyway). Needs a MaxMind license key to provision:
        // stage.geo_ip {
        //   db           = "/var/lib/geoip/GeoLite2-City.mmdb"
        //   source       = "client_ip"
        //   target_label = "geo_country"
        // }
      }
    }

    loki.write "default" {
      endpoint {
        url = "http://127.0.0.1:3100/loki/api/v1/push"
      }
    }
  '';
}
```

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
