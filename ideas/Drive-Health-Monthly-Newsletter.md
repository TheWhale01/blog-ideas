---
title: Drive Health Monthly Newsletter
created: 2026-11-07
updated: 2026-11-07
type: idea
tags: [erebos, homelab, monitoring, IT]
sources: []
---

# Drive Health Monthly Newsletter

**Status**: feasible — unblocked (prod is bare metal; SMART is local)

## Goal

A **monthly email newsletter** on the state of the physical hard drives ("should I replace one?"),
sourced from the **existing monitoring stack** — so the same data is also always live on a Grafana
dashboard, no need to wait for the mail to check.

## Feasibility verdict — YES, fully local

**Erebos prod is a physical machine** (owner-confirmed 2026-11-07; verified live:
`systemd-detect-virt` = none, AMD Ryzen 5 5600, six ATA disks visible in `/sys/block`). SMART data is
therefore directly collectable **on Erebos itself** — no host/VM indirection. (Erebos *hosts* the
erebos-stage VM via libvirtd; the stage environment is deliberately out of scope for the newsletter —
its virtual disks are portable by design.)

The "array" — the actual physical inventory, verified 2026-11-07:

| Device | Disk | Role hint |
|---|---|---|
| `sda` | WDC WD20EARX (2 TB, Green) | |
| `sdb` | WDC WD6400AAKS (640 GB) | |
| `sdc` | WDC WD10EZEX (1 TB) | |
| `sdd` | Kingston SA400S3 (SATA SSD) | **root: ESP + btrfs** (disko `nixos` disk, corrected `6973308`) |
| `sde` | WDC WD1001FALS (1 TB Black) | |
| `sdf` | Seagate ST8000DM004 (8 TB BarraCuda) | likely `/data` (VG `whale`) — verify with `pvs` |

No mdraid/ZFS — redundancy does not exist; SMART-driven early replacement is the *only* failure
strategy, which makes this idea more load-bearing than it looks. Two of these drives (the 640 GB
AAKS, the 2011-era 1001FALS) are old enough that power-on hours alone may decide the first swap.

```
Erebos (bare metal) ── smartctl_exporter (:9633, new) + node_exporter (:9100, exists)
                          │ scraped locally
                    Prometheus ── Grafana "Disk Health" dashboard   ← always-viewable
                          │
              Hermes monthly cron ── Prometheus HTTP API → digest → Resend email → inbox
```

## Components (researched & verified 2026-11-07)

1. **Collection** — [smartctl_exporter](https://github.com/prometheus-community/smartctl_exporter)
   on Erebos itself via the NixOS module [`services.prometheus.exporters.smartctl`](https://search.nixos.org/options?query=services.prometheus.exporters.smartctl)
   (wraps [smartmontools](https://www.smartmontools.org/)). Exposes per-disk: health status,
   reallocated/pending/uncorrectable sectors, temperature, power-on hours, SSD wear (percentage used),
   test results. Bind to `localhost:9633`, scrape locally — the loopback rule from
   [[queries/security-audit]] applies (nothing needs this over the LAN).
   *Caveat found:* SATA/HDD coverage is rock-solid; the SATA SSD (Kingston) reports vendor-attribute
   style wear — verify its `percentage_used` equivalent appears; NVMe/RAID-card issues don't apply here.
   Plus `node_exporter` (already running) for I/O error counters; no mdstat/ZFS collector needed —
   there is no mdraid or pool, just ATA disks + LVM.
   Bonus, same machine, complementary: enable `services.smartd` (smartmontools daemon) to run weekly
   SHORT / monthly LONG self-tests and mail/report failures — the newsletter *reads* trends, smartd
   *creates* them.

2. **Always-on view** — scrape `localhost:9633` from Erebos' Prometheus (one extra job in
   `sys/services/monitoring/`), then a Grafana dashboard wired to the `smartctl_*` metric names —
   per-disk tiles: health flag, temp, reallocated/pending sectors, power-on hours, SSD wear.
   *Alternative single-app:* [Scrutiny](https://github.com/AnalogJ/scrutiny) — a self-contained SMART
   web UI + daemon with its own dashboards. Cleaner as a one-container answer, but it duplicates what
   Prometheus already gives you and sits outside the alerting plan; documented, not recommended here.

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
   (Scrutiny-style all-in-one setups mail hourly themselves, but bypass Grafana/alerting.)

## Remaining open questions (no longer gating — sizing questions)

1. Newsletter content depth: pure SMART digest, or include filesystem usage + backup-job status too
   (a true "state of the drives AND their data" monthly)?
2. Recipient/route: personal inbox via Resend (token reuse) — confirm the from-address policy
   (`authentik-no-reply@thewhale.fr` exists; a dedicated `odin@` might be cleaner).

## Tasks

- [ ] Enable `services.prometheus.exporters.smartctl` (localhost bind) + scrape job + `services.smartd` self-tests in `sys/services/monitoring/`
- [ ] Finish the disk-role map (`sudo pvs` → which disk hosts VG `whale`; identify sda/sdb/sdc/sde roles) and optionally move disko to stable `/dev/disk/by-id/ata-…` paths (root is now correctly `/dev/sdd` per `6973308`)
- [ ] Build the Grafana disk-health dashboard (tiles per `/sys/block` disk, thresholds colored)
- [ ] Hermes monthly cron digest: Prometheus queries → green/yellow/red rubric → Resend email + vault archive
- [ ] Immediate Alertmanager rules via [[todo/monitoring-alerting]] (`smartctl_health`, sector growth, temp)
- [ ] Define replace-verdict thresholds (new reallocated/pending > 0 sustained; wear > 80 %; SMART self-test errors; power-on > ~40k h on greens)
- [ ] First digest = manual dry-run, cross-checked against `sudo smartctl -a /dev/sdX` on each disk

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
