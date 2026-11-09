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
| `promtail.nix` | *currently* Alloy | Log shipping — file is fully commented-out Promtail; only `services.alloy.enable = true` with **no settings** → **nothing is being collected, Loki holds zero series** (verified 2026-11-07) |

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
