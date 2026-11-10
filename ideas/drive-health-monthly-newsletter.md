---
title: Drive Health Monthly Newsletter
created: 2026-11-07
updated: 2026-11-07
type: idea
tags: [erebos, homelab, monitoring, IT]
sources: []
---

# Drive Health Monthly Newsletter

**Status**: 📬 NEWSLETTER LIVE (cron + issue 1) on top of the tripwire.
**Status**: 🔴 TRIPWIRE PROVED ITSELF — first smartd boot (2026-11-10 22:12) wall-alerted `sdb`:
207 pending + 190 uncorrectable sectors (drive ~2009). Pool still reads clean (no kernel I/O errors
3h window; exporter not yet up — smart.nix unpushed/partial apply). Playbook below.

**Status**: tripwire half DELIVERED + VALIDATED 2026-11-10 (snippets in chat, owner applies);
newsletter half still queued behind it.
Validation performed: `eval-config` of the exact snippets against pinned nixpkgs (options all exist,
assertions pass, rendered `smartd.conf` = DEFAULT `-a -o on` + 6 staggered device lines);
prometheus-community smartctl_exporter 0.14.0 ExecStart binds `127.0.0.1:9633` @ interval 5m;
`smartctl_device_smart_status` semantics source-confirmed (1=healthy, 0=FAILED) so `== 0` is the
tripwire; drive_health group (5 rules) merged → `promtool check` 17 SUCCESS + existing regression
suite still green; dashboard 22381 verified metric-compatible (its `DS_VICTORIAMETRICS` ×30
placeholders handled by terraform `replace()` exactly like the traefik 17346 wiring).

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

| Device | Disk | Role (pvs-verified 2026-11-07) |
|---|---|---|
| `sda` | WDC WD20EARX (2 TB, Green) | PV of `whale-nas` (<1.82 t) |
| `sdb` | WDC WD6400AAKS (640 GB, ~2009-era) | PV of `whale-nas` via `sdb3` (595 g) — 🔴 **SMART FAILING (2026-11-10): 207 pending + 190 offline-uncorrectable sectors — replacement triggered** |
| `sdc` | WDC WD10EZEX (1 TB) | PV of `whale-nas` via `sdc1` |
| `sdd` | Kingston SA400S3 (SATA SSD) | **root: ESP + btrfs** (disko `nixos` disk, corrected `6973308`) — only disk outside the pool |
| `sde` | WDC WD1001FALS (1 TB Black, ~2011-era) | PV of `whale-nas` via `sde1` — ⚠️ old |
| `sdf` | Seagate ST8000DM004 (8 TB BarraCuda) | PV of `whale-nas` via `sdf1` (7.28 t) — SMR drive ⚠️ |

**No mdraid, no ZFS — and `/data` is a linear LVM pool across five disks with ~0 free extents:**
one failure domain, zero redundancy, and no spare room to pvmove around a dying member. The pool's
availability is the *product* of its five members' health (plus the two ancient ones are likely to
set the pace) — this is precisely the math the newsletter exists to watch. SMART-driven early
replacement is the **only** failure strategy; add the SMR caveat (BarraCuda 8 TB: unsuitable for
heavy random-write workloads; worth checking `smartctl` wear/idle-cick counts monthly) and note the
VG-full condition implies any replacement = add a new disk first, then pvmove off the old one.

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

- [x] ~~Enable exporters.smartctl + scrape job + smartd self-tests~~ — snippets delivered &
      eval-validated 2026-11-10: `sys/services/monitoring/smart.nix` (smartd: autodetect=false, 6
      devices with staggered selftests — daily shorts 00–04h, weekly longs Sat 03–07h, SMR 8 TB long
      moved to WED so it never overlaps the Sunday borg window; mail stays off (no MTA), wall
      notification on = free console alert; journal output → Loki automatically as `{unit="smartd.service"}`)
      + scrape job `smartctl` + `smartd` added to `watchedServices`. Note: `prometheus-smartctl-exporter.service`
      is ALREADY covered by the existing `prometheus` whitelist entry + the `.*\.service` suffix regex.
- [ ] Optional: move disko to stable `/dev/disk/by-id/ata-…` paths (root `sdd` + whale-nas member layout now fully mapped via `pvs`)
- [ ] Check `sdb1/sdb2` — the 640 GB has two partitions before `sdb3`; identify leftovers, don't confuse the disk map
- [ ] Dashboard: terranix wiring delivered (data.http 22381 + `replace(... DS_VICTORIAMETRICS → vars.prometheus.uid)`);
      owner applies + sanity-check the Kingston SSD wear panel (vendor-attr naming).
- [x] ~~Hermes monthly cron digest~~ — LIVE 2026-11-10: cron job `drive-health-monthly-newsletter`
      (id 21ce6f527471, `0 9 1 * *` Europe/Paris, workdir = vault, delivers to the owner DM).
      Prompt embeds: rubric incl. WD-health-byte lie guard, Kingston vendor-attr fallback, RFC3339
      Loki windows, read-only loopback rule, prepend-issue-and-push archive flow.
      Issue 1 (baseline) hand-generated first: [[queries/drive-health]] (sdb RED — first-ever
      self-test failed same night, self-test errors 0→3). Email delivery via Resend NOT wired
      (Matrix delivery chosen; email optional add-on later).
- [x] ~~Immediate Alertmanager rules~~ — delivered as `drive_health` group (promtool-validated):
      DiskSmartFailing (`smartctl_device_smart_status == 0`, critical), DiskReallocatedSectors /
      DiskPendingSectors (`> 0` raw, 30m warning), DiskUncorrectableSectors (`Offline_Uncorrectable > 0`
      critical), DiskTooHot (>55°C). Power-on-hours intentionally NOT an alert (static gauge =
      permanent fire) — that verdict belongs in the newsletter rubric.
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

## Replacement playbook — sdb (triggered 2026-11-10)

1. **Assess** (host): `sudo smartctl -x /dev/sdb` (Reallocated count? Self-test log?) +
   `sudo smartctl -t long /dev/sdb` (~1 h; more bad regions surface — informational, drive stays usable during test).
   `sudo journalctl -k --since '-30 days' | grep -iE "I/O error|medium error|blk_update"` — historical read failures?
2. **Verify backup NOW** (don't wait for Sunday's borg): run the borg job manually and read its
   per-file errors — files it cannot read ARE the sectors already dead. This answers "did we lose
   data yet?" definitively.
3. **Pool has ~0 free extents** (verified 2026-11-07) ⇒ cannot pvmove off a dying disk without more
   space: add replacement disk first (any SATA ≥ 1 TB preferred, ≥ 640 GB minimum) →
   `pvcreate` → `vgextend whale-nas` → `pvmove --background /dev/sdb3` → (if pvmove aborts on an
   unreadable extent: that file is restored from borg after the move) → `vgreduce whale-nas /dev/sdb3`
   → `pvremove` → power off the disk.
4. **Alerts**: `DiskPendingSectors` (warn) + `DiskUncorrectableSectors` (crit) will fire once the
   exporter runs and rules evaluate — legit noise until sdb is gone; silence in the Alertmanager UI
   if the 4h repeat gets loud, don't delete the rules.
5. **Follow-up**: sde (WD1001FALS, ~2011) is the next-priority candidate; power-on hours will show
   it. The two Greens' spin-down behavior vs SMART temp readings — check after exporter is live.

## Open defect (2026-11-10)

smart.nix not pushed; applied state has smartd running but **prometheus-smartctl-exporter absent**
(no unit, no :9633, no journal lines) — exporter stanza missing or service failed. Needs:
`systemctl status prometheus-smartctl-exporter` + push the file; then scrape job + rules + dashboard
can be validated (17-rule set already promtool-verified pending apply).

## First-strike postmortem — sdb confirmed dead-on-feet (2026-11-10, first 45 min of the pipeline)

Full fleet read (exporter `9633` up 22:21, 17 rules loaded, up=1):

| disk | POH | realloc | pending | uncorr | verdict |
|---|---|---|---|---|---|
| sda WD20EARX | 27,518 h | 0 | 0 | 0 | 🟢 healthy |
| sdb WD6400AAKS-41H2B0 (Caviar **Blue**) | 39,016 h | **280** (160 events) | **207** | **190** | 🔴 replace now — error log 556 events, latest cluster same LBA |
| sdc WD10EZEX | 56,074 h | 0 | 0 | 0 | 🟡 oldest runtime, clean — watch, 2nd replacement candidate |
| sdd Kingston SA400S3 (root) | 19,709 h | 0 | 0 | 0 | 🟢 life-left 67 %, bad-block/program/erase all 0; Nov-9 kernel I/O error = ONE SATA PHY/CRC blip (link, not media) — reseat cable |
| sde WD1001FALS | 49,847 h | 0 | 0 | 0 | 🟡 old + **UDMA_CRC=41** (flaky cable → fake deaths) — reseat/swap cable €5 |
| sdf ST8000DM004 8 TB SMR | (packed attr — see note) | 0 | 0 | 0 | 🟢 counters clean; POH via `smartctl -A /dev/sdf` (exporter float mangles Seagate packed attrs — `1.08e16` = garbage) |

**Key lessons (documented as pitfalls):**
1. **WD health-byte lie:** `smartctl_device_smart_status` reports PASSED (1) for sdb despite everything —
   older WDs never assert FAILED. `DiskSmartFailing` is decorative on this fleet; the sector-attribute
   rules + smartd wall are the real tripwires (both fired today within 90 s of first boot). Do not
   relax the sector rules on the strength of a PASSED.
2. The 190 Offline_Uncorrectable likely surfaced TODAY because smartd `-o on` started Auto Offline
   Data Collection (SCT state confirms it executing) — the drive had been hiding them; `-t long`
   (started 22:25, completes Wed 00:43, 136 min) will enumerate the rest. Check
   `sudo smartctl -l selftest /dev/sdb` after 00:43 — expect possibly more pending→reallocated churn.
3. Packed-attribute vendors (Seagate) need `smartctl` CLI truth for POH — exporter float64 loses low bits.

**Actions pending owner (tonight/this week):**
- [ ] ~22:54: three Matrix alerts expected on sdb (uncorrectable crit + pending + realloc) — acknowledge, silence repeat until resolved.
- [ ] Run the borg job NOW if deployed (else: this week's data-safety headline is "borg not yet live" —
      verify what on the pool is actually backed up; immic/nextcloud/vaultwarden are the borg scope,
      bulk media is single-copy and includes any file living on sdb's dead LBAs).
- [ ] Order replacement SATA ≥ 1 TB (suggest TWO: sdb now; sde pre-emptive — 50k h + 41 CRC events).
- [ ] Reseat SATA cables sdd + sde while case open (both show link errors, not media errors).
- [ ] After 00:43: paste `smartctl -l selftest /dev/sdb` tail → update pending/uncorrectable deltas here.
- [ ] Push `smart.nix` + `config.nix` rules additions — production is running SMART config that exists
      only in your working tree right now (repo tip was 98a56d4 at 22:30).
