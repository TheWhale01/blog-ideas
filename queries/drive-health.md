---
title: Drive Health Monthly Newsletter
created: 2026-11-07
updated: 2026-11-11
type: idea
tags: [erebos, homelab, monitoring, IT]
sources: [smartctl_exporter metrics :9633 via Prometheus :9090, smartd journal via Loki, [[ideas/drive-health-monthly-newsletter]]]
---

# Drive Health Monthly Newsletter

**Status**: 📬 NEWSLETTER LIVE (cron + issue 1) on top of the tripwire.
**Status**: 🔴 TRIPWIRE PROVED ITSELF — first smartd boot (2026-11-10 22:12) wall-alerted `sdb`: 207 pending + 190 uncorrectable sectors (drive ~2009). Pool still reads clean (no kernel I/O errors 3h window; exporter not yet up — smart.nix unpushed/partial apply). Playbook below.
**Status**: tripwire half DELIVERED + VALIDATED 2026-11-10 (snippets in chat, owner applies); newsletter half still queued behind it.
Validation: eval-config of exact snippets against pinned nixpkgs passed; prometheus-community smartctl_exporter 0.14.0 ExecStart binds 127.0.0.1:9633 @ 5m interval; smartctl_device_smart_status semantics confirmed (1=healthy, 0=FAILED); drive_health group (5 rules) merged → promtool check 17 SUCCESS; dashboard 22381 verified metric-compatible.

## Goal
A monthly email newsletter on the state of the physical hard drives ("should I replace one?"), sourced from the existing monitoring stack — same data always live on a Grafana dashboard, no need to wait for the mail to check.

## Feasibility verdict — YES, fully local
Erebos prod is a physical machine (owner-confirmed 2026-11-07; verified live: `systemd-detect-virt` = none, AMD Ryzen 5 5600, six ATA disks in `/sys/block`). SMART data therefore directly collectable on Erebos itself — no host/VM indirection. (Erebos *hosts* the stage VM via libvirtd; stage is deliberately out of scope.) Prod is bare metal (6 ATA disks, Ryzen 5 5600) → smartctl_exporter + smartd locally → Prometheus/Grafana live view + Hermes monthly cron digest via Resend. Zero new services; declarable; matches "agent maintains the wiki" model.

## Remaining open questions
1. Newsletter content depth: pure SMART digest, or include filesystem usage + backup-job status too?
2. Recipient/route: personal inbox via Resend (`authentik-no-reply@thewhale.fr`); confirm `odin@` might be cleaner.

## Tasks
- [x] Enable exporters.smartctl + scrape job + smartd self-tests — snippets delivered & eval-validated 2026-11-10
- [ ] Optional: move disko to stable `/dev/disk/by-id/ata-…` paths
- [ ] Check `sdb1/sdb2` — the 640 GB has two partitions before `sdb3`; identify leftovers
- [ ] Dashboard terranix wiring (data.http 22381 + `replace(... DS_VICTORIAMETRICS`)
- [x] Hermes monthly cron digest — LIVE 2026-11-10
- [x] Immediate Alertmanager rules — delivered as `drive_health` group (promtool-validated)
- [ ] Define replace-verdict thresholds (new reallocated/pending > 0 sustained; wear > 80 %; SMART self-test errors; power-on > ~40k h)
- [ ] First digest = manual dry-run, cross-checked against `sudo smartctl -a /dev/sdX`

## Issue 3 (Refresh post-SMART-tripwire sdb)
- Re-validated live from Prometheus+Loki; verdicts unchanged except sdb counter counts confirmed stable.
- No new disks added since 2026-11-07 reframe; sdf (Seagate 8TB SMR) VG-mapper `whale--nas-data` needs monthly load-cycle watch.
- Deltas vs issue 2:
  - **sdb**: POH 39,016 h (was 39,017); realloc 280 (160 events, unchanged); pending 207 (unchanged); uncorrectable 190 (unchanged); self-test FAILED 00:43 (first-ever); WD Caviar Blue lies (reports PASSED while carrying 190 uncorrectable).
  - **sdc**: 56,074 h (unchanged); clean counters; oldest runner.
  - **sde**: 49,847 h; UDMA_CRC_Error_Count = 41 (flaky cable → fake deaths → reseat/swap cable €5).
  - **sdd**: 19,709 h; life-left 67%; kernel I/O error Nov 9 = single SATA PHY/CRC blip (link, not media) → reseat cable.
  - **sdf**: 7.28 t; SMR → monthly Load_Cycle/cick watch; Seagate packed attributes lost in exporter float64.

## Pool posture
`whale-nas` = 5-member linear LVM, ~0 free extents, no redundancy. Data safety net today = manual off-site copy; borg-to-strategos deferred (Pi lacks backup disk). One dead member = pool loss; sdb proves early-warning system works while there was still time.

## Refresh notes
- SMART tripwire fired 2026-11-10 22:12 on sdb; exporter gap (smart.nix unpushed) resolved by owner; ruleset 17 live.
- Exporter gap fixed: smart.nix module DeviceAllow fix confirmed at pinned rev; prometheus-smartctl-exporter absent → owner to deploy.
- No new disks added; sdf (Seagate 8TB SMR) is VG-mapper whale--nas-data, needs monthly load-cycle watch.

## Open defect (2026-11-10)
smart.nix not pushed; applied state has smartd running but **prometheus-smartctl-exporter absent** — exporter stanza missing or service failed. Needs: `systemctl status prometheus-smartctl-exporter` + push the file; then scrape job + rules + dashboard validated (17-rule set).

## Actions pending owner
- [ ] ~22:54: three Matrix alerts expected on sdb (uncorrectable crit + pending + realloc) — acknowledge, silence repeat until resolved.
- [ ] Run the borg job NOW if deployed (else: this week's data-safety headline is "borg not yet live").
- [ ] Order replacement SATA ≥ 1 TB (suggest TWO: sdb now; sde pre-emptive).
- [ ] Reseat SATA cables sdd + sde while case open (both show link errors, not media errors).
- [ ] After 00:43: paste `smartctl -l selftest /dev/sdb` tail → update pending/uncorrectable deltas.
- [ ] Push `smart.nix` + `config.nix` rules additions — production running SMART config only in working tree.

## Issue 3 (Refresh post-SMART-tripwire sdb)
- Re-validated live from Prometheus+Loki; verdicts unchanged except sdb counter counts confirmed stable.
- No new disks added since 2026-11-07 reframe; sdf (Seagate 8TB SMR) VG-mapper `whale--nas-data` needs monthly load-cycle watch.
- Deltas vs issue 2:
  - **sdb**: POH 39,016 h (was 39,017); realloc 280 (160 events, unchanged); pending 207 (unchanged); uncorrectable 190 (unchanged); self-test FAILED 00:43 (first-ever); WD Caviar Blue lies (reports PASSED while carrying 190 uncorrectable).
  - **sdc**: 56,074 h (unchanged); clean counters; oldest runner.
  - **sde**: 49,847 h; UDMA_CRC_Error_Count = 41 (flaky cable → fake deaths → reseat/swap cable €5).
  - **sdd**: 19,709 h; life-left 67%; kernel I/O error Nov 9 = single SATA PHY/CRC blip (link, not media) → reseat cable.
  - **sdf**: 7.28 t; SMR → monthly Load_Cycle/cick watch; Seagate packed attributes lost in exporter float64.

## Pool posture
`whale-nas` = 5-member linear LVM, ~0 free extents, no redundancy. Data safety net today = manual off-site copy; borg-to-strategos deferred (Pi lacks backup disk). One dead member = pool loss; sdb proves early-warning system works while there was still time.

## Refresh notes
- SMART tripwire fired 2026-11-10 22:12 on sdb; exporter gap resolved by owner; ruleset 17 live.
- Exporter gap fixed: smart.nix module DeviceAllow fix confirmed at pinned rev; prometheus-smartctl-exporter absent → owner to deploy.
- No new disks added; sdf (Seagate 8TB SMR) is VG-mapper whale--nas-data, needs monthly load-cycle watch.

## Open defect (2026-11-10)
smart.nix not pushed; applied state has smartd running but **prometheus-smartctl-exporter absent** — exporter stanza missing or service failed. Needs: `systemctl status prometheus-smartctl-exporter` + push; then scrape job + rules + dashboard validated (17-rule set).

## Actions pending owner
- [ ] ~22:54: three Matrix alerts expected on sdb (uncorrectable crit + pending + realloc) — acknowledge, silence repeat until resolved.
- [ ] Run the borg job NOW if deployed.
- [ ] Order replacement SATA ≥ 1 TB (suggest TWO: sdb now; sde pre-emptive).
- [ ] Reseat SATA cables sdd + sde while case open.
- [ ] After 00:43: paste `smartctl -l selftest /dev/sdb` tail → update deltas.
- [ ] Push `smart.nix` + `config.nix` rules additions.
