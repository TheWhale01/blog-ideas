---
title: Monitoring
tags:
  - nixos
  - erebos
  - monitoring
  - observability
---

# Erebos — Monitoring Stack

The monitoring stack lives in `sys/services/monitoring/` and is composed by `sys/services/monitoring/default.nix`.

## Components

| File | Tool | Role |
|---|---|---|
| `prometheus.nix` | Prometheus | Time-series metrics and alerting rules |
| `grafana.nix` | Grafana | Dashboards, visualization, alerting UI |
| `loki.nix` | Loki | Log aggregation |
| `promtail.nix` | Promtail | Log shipping (collects logs and ships to Loki) |

## Integration

The monitoring stack is wired into the NixOS config through:

- `sys/services/monitoring/default.nix` (imports all four modules);
- `sys/services/default.nix` (the monitoring module is imported here);
- Traefik dynamic config (Grafana UI exposed at `grafana.thewhale.fr`, behind Authentik).

## TODO

From the original `Erebos` note: Grafana + Prometheus are deployed, but **no alerting is running** and a proper log analyzer is missing. The TODO list includes properly configuring alerting (e.g. Grafana Alerting / Alertmanager) and a solid log-analysis pipeline.

## External Monitoring

Because Erebos is a server, it also relies on:

- **Tailscale** (`sys/services/tailscale.nix`) — VPN for safe remote management;
- **Traefik's** own `metrics.prometheus` endpoint (exposed on `127.0.0.1:8200`).

These are all part of the overall monitoring and observability picture.
