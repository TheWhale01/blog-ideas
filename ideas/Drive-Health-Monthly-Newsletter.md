---
title: Drive Health Monthly Newsletter
created: 2026-11-07
updated: 2026-11-07
type: idea
tags: [erebos, homelab, monitoring, IT]
sources: []
---

# Drive Health Monthly Newsletter

**Status**: researched — feasible, with one location decision to make first

## Goal

A **monthly email newsletter** on the state of the physical hard drives ("should I replace one?"),
sourced from the **existing monitoring stack** — so the same data is also always live on a Grafana
dashboard, no need to wait for the mail to check.

## Feasibility verdict — YES, with one structural requirement

**Erebos is a VM** (`/dev/vda` virtio root, `libvirtd` on the physical machine beneath).
SMART is a property of *physical* disks and **does not cross a virtio/VM boundary** — `smartctl`
inside Erebos sees nothing useful about the real drives. Therefore the collection layer must run on
**the physical host that owns the disks** (the box Erebos's VMs run on — the machine holding the
array in question). Everything else slots into the stack that already exists:

```
physical host ── smartctl_exporter (:9633) + node_exporter (:9100)
                     │  (scraped over the LAN by…)
Erebos ── Prometheus ─┤── Grafana dashboard  ← always-viewable state ("so I don't need to wait")
                     │
        monthly job ──┘── query Prometheus HTTP API → compose mail → Resend SMTP → inbox
```

## Components (researched & verified 2026-11-07)

1. **Collection** — [smartctl_exporter](https://github.com/prometheus-community/smartctl_exporter)
   (prometheus-community; a NixOS module exists: [`services.prometheus.exporters.smartctl`](https://search.nixos.org/options?query=services.prometheus.exporters.smartctl),
   wraps [smartmontools](https://www.smartmontools.org/)). Exposes per-disk: health status, reallocated/pending/
   uncorrectable sectors, temperature, power-on hours, SSD wear (percentage used), smartd test results.
   Runs on the disk host; NVMe needs the device nodes readable (`disk` group / udev rule per the module docs).
   *Caveat found:* SSD/NVMe coverage is best-effort — some vendor attributes surface inconsistently;
   classic SATA/HDD metrics are solid.
   Plus `node_exporter` (already the fleet pattern) for I/O errors, and its built-in **mdstat** collector
   if the array is mdraid — or a ZFS pool exporter if the host is TrueNAS/Proxmox-ZFS (→ open question below).

2. **Always-on view** — scrape `host:9633` + `host:9100` from Erebos' Prometheus (add one job to
   `sys/services/monitoring/`), then a Grafana dashboard: the community "SMART / Disk Health" panels
   plug straight into `smartctl_*` metric names. Note: **Grafana must reach the host from Erebos** —
   the exporter binds the LAN interface; keep it firewalled to the Erebos VM IP (never public; the
   loopback-bindings lesson from [[queries/security-audit]] applies — bind `--web.listen-address=host-virbr-ip:9633`
   or firewall-source-restrict).
   *Alternative single-app:* [Scrutiny](https://github.com/AnalogJ/scrutiny) — a self-contained SMART
   web UI + daemon on the host (Docker or NixOS service) with its own dashboard and (older) built-in
   hourly email. Cleaner if the host isn't NixOS; duplicates Prometheus-side features though.

3. **The newsletter itself** — Grafana OSS **has no native scheduled email/PDF reporting** (it's an
   Enterprise/Cloud feature — [docs](https://grafana.com/docs/grafana/latest/dashboards/create-reports/)),
   so choose a delivery mechanism:
   - **A. Hermes monthly cron job (recommended):** exactly the [[todo/scheduled-security-audit]] pattern
     — a monthly scheduled prompt: query Prometheus' HTTP API (`smartctl_*` + filesystem/device metrics),
     render a compact digest (per-disk: health, attrs deltas vs last month, temp, power-on hours, a
     computed **green/yellow/red** verdict with the classic replace triggers: any new reallocated/
     pending sectors, wear % thresholds, SMART self-test failures, age > ~5y/40k h), email it via
     **Resend** (the API token already exists on the box inside `authentik-smtp.age` — copy it into
     `hermes.age` so the agent can call the API), push the same digest
     into `queries/` or the Matrix room as a side effect so history accumulates in the vault.
     *Zero new services; declarable; matches the "agent maintains the wiki" model.*
   - **B. n8n workflow:** Schedule → HTTP (Prometheus) → Email node. Already-running tool, UI-managed
     (less declarative).
   - C. Paid/3rd-party report tools (Skedler etc.) — overkill; skip.

4. **Do NOT make "monthly" the only feedback loop** — a drive going SMART-FAILED mid-month shouldn't
   wait for the newsletter. The same `smartctl_health` metric feeds instant Alertmanager rules
   (`smartctl_health == 0`, pending-sectors jump, temp > 55 °C) → Matrix. That's a sub-item of
   [[todo/monitoring-alerting]]; the newsletter is the *trend* view, alerts are the *tripwire*.
   (If the host runs Scrutiny instead, it can also mail hourly — but then it bypasses Grafana.)

## Open question that gates the build

**What is the physical host, and can we deploy to it?** The plan assumes a Linux box we control
(NixOS ideal → the exporter modules drop in next to `sys/services/monitoring/` style configs; any
Linux → static binary + systemd unit; TrueNAS/Proxmox → exporter story changes: Proxmox has native
SMART + its own metrics API; TrueNAS SCALE runs k3s/docker → Scrutiny fits there). "My array" needs
a definition: mdraid? ZFS pool? plain passthrough disks into the VM? → determines which collectors
besides SMART matter (mdstat / zpool status / btrfs device stats).

## Tasks

- [ ] Answer the host question (OS + how the disks are attached + what "the array" actually is)
- [ ] Deploy `smartctl_exporter` (+ `node_exporter` if missing) on the host, restricted bind/firewall
- [ ] Scrape job in Erebos' Prometheus + Grafana disk-health dashboard (verify live view first)
- [ ] Hermes monthly cron job: Prometheus query → digest → Resend email (+ archive to vault)
- [ ] Immediate-alert rules via [[todo/monitoring-alerting]] (`smartctl_health`, sector growth, temp)
- [ ] Define the per-drive replace verdict thresholds in the prompt (green/yellow/red rubric)
- [ ] First newsletter = manual dry-run, sanity-check the numbers against `smartctl -a` on the host

## Related

- [[docs/erebos/monitoring]] — the Prometheus/Grafana/Loki stack this extends
- [[todo/monitoring-alerting]] — the tripwire complement
- [[todo/scheduled-security-audit]] — the same agent-cron mechanism as the digest
- [[docs/strategos/reference]] — wake-on-LAN helper; strategos survives Erebos-down (backup mail target?)

## External references

*(all curl-verified 2026-11-07)*

- **smartctl_exporter (Prometheus)** — https://github.com/prometheus-community/smartctl_exporter
- **NixOS module option** — https://search.nixos.org/options?query=services.prometheus.exporters.smartctl
- **smartmontools (smartctl/smartd)** — https://www.smartmontools.org/
- **Scrutiny (alternative all-in-one disk UI)** — https://github.com/AnalogJ/scrutiny
- **Grafana reporting docs (Enterprise-only, hence the DIY mail)** — https://grafana.com/docs/grafana/latest/dashboards/create-reports/
- **Resend (SMTP/API already used for Authentik mail)** — https://resend.com/docs
