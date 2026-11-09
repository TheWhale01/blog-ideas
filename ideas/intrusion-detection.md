---
title: Intrusion Detection on Erebos
created: 2026-11-07
updated: 2026-11-07
type: idea
tags: [erebos, security, loki, alloy, auditd, detection]
sources: ["owner question 2026-11-07", "live stack assessment 2026-11-07"]
---

# Intrusion Detection on Erebos

> Goal: see intrusion **attempts**, know whether they **succeeded**, and make the evidence
> **tamper-resistant**. Status: research — the existing stack *architecturally* supports this,
> but is currently blind (see assessment).

## Assessment of the current stack (verified 2026-11-07)

The pieces exist — Prometheus (4 scrape jobs incl. `authentik` + `traefik` metrics, `tcpstat`
collector on), Loki, Grafana, Alertmanager→Matrix. But the **log pipeline is a facade today**:

- `promtail.nix`: whole config commented out; replaced by `services.alloy.enable = true` **with no
  `settings`** → Alloy runs an empty graph and collects nothing
- Loki `/loki/api/v1/label` returns **zero labels** — nothing has ever been ingested
- Loki **ruler not configured** → logs can't alert anyway
- ⚠️ Preconditions, both already tracked: `auth_enabled = false` on a Loki listening on `*:3100`
  means an attacker (or any container on trusted `podman0`) can **read and DELETE logs via the API**
  — plus the P2 loopback binds from [[queries/security-audit]]

Answer to "is it possible?": **yes, ~80% of the machinery is installed; what's missing is feeding
it and pointing its alerts at Alertmanager.**

## Signal layers (in build order)

1. **Journal → Loki** (one config, instant coverage of sshd, sudo, kernel, every unit):
   Alloy [`loki.source.journal`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.journal/)
   + [`loki.write`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.write/) →
   localhost Loki. (Or simply un-comment promtail — it had journal + traefik geoip enrichment drafted.)
   Alloy needs the service user in the `systemd-journal` group (the commented `extraGroups` line
   was the same idea for promtail).
2. **Traefik access log** (edge attempts: 404-hunting, auth floods, odd user-agents, geoip
   anomalies via the drafted MaxMind pipeline) — journal source covers `traefik.service` already.
3. **Authentik = auth ground truth** (the "did they succeed?" oracle): its Prometheus events are
   ALREADY scraped — alert on `increase(authentik_events{action="login",is_successful="false"}[10m])`
   spikes; a *successful* login from a new geo/IP after failures is the worst-case line — pattern in
   the journald stream once layer 1 lands.
4. **Loki ruler alerts** ([docs](https://grafana.com/docs/loki/latest/alert/)) — the log-side
   [[todo/monitoring-alerting]] companion, rules fed to the SAME Alertmanager (single brain):
   - `{job="journal",unit="sshd.service"} |~ "Failed password" | pattern <ip> | count > 10 [5m]`
   - `{job="journal",unit="sshd.service"} |~ "Accepted publickey|Accepted password"` — INFO card to
     the ops room (every real session = audit trail, cheap and brutal: works)
   - `sudo: a session opened` lines, authentik container/podman OOMs, etc.
   - **Pipe-failure alarm** (tamper/availability): rate of journal lines drops to ~0 for 15 m →
     `ScrapeTargetDown`-equivalent for logs. If an attacker kills observability, it pages.
5. **auditd** ([wiki](https://github.com/Linux-Audit-Project/linux-audit/wiki) /
   [audit-userspace](https://github.com/linux-audit/audit-userspace),
   [rules syntax](https://manpages.debian.org/bookworm/auditd/audit.rules.7.en.html)) — syscall-level
   truth journald doesn't see: watch `useradd`/`passwd`/shadow changes, writes to
   `~/.ssh/authorized_keys`, `ld.so.preload`, execs from `/tmp`, `nft`/`iptables` mutations.
   `services.auditd.enable` on NixOS; audit events land in journald → layer 1 ships them to Loki.
   Keep ruleset surgical (file-watches + execve-on-interesting), a full syscall audit burns the Pi's
   successor… burns *disk* on this server.

## The NixOS cheat-code: drift becomes trivially small

On immutable NixOS the OS itself is content-addressed — **an intruder's persistence can only live in
`/etc`, `/var/lib`, `/data`, `/home`, or a new unit** (and store paths, which are hash-verified).
That collapses "file integrity monitoring" to a tiny, loud watch-set:

- `nix path-verifier`: `nix store verify --all` as a weekly job (corrupt/altered store = game over anyway)
- new systemd units appearing: `count(node_systemd_unit_state)` jump detection (metrics already exist!)
- auditd watches above; mtime deltas on `/etc/ssh`, `/etc/nixos`, authorized_keys
- **The elegant one**: `configuration-rollback-detected` — compare `/run/current-system` symlink hash
  against the value recorded in Prometheus as a metric/gauge; any activation is expected & logged,
  unannounced drift = 🚨

## Tamper-resistance (the "succeeded or not" must survive root loss)

- Ruler alerts already leave the box instantly via [[todo/monitoring-alerting]] → Matrix (best
  property of the whole design: attacker-with-root can silence the box, not the room).
- Loki stays behind **loopback + auth** (or Traefik+Authentik in front if remote viewing wanted).
- Option: ship compact daily journald snapshots (`journalctl --since … --vacuum-free export`) into
  the **append-only borg repo on strategos** → history survives full disk wipe of Erebos; retention
  2–4 weeks is plenty for incident response at homelab scale.
- Off-site Loki (strategos) — possible but Pi 3B+ RAM (~1 GB) makes a second Loki+Alloy pipeline
  tight; the borg-snapshot route gets 90% of the forensic value at 10% of the complexity.

## Active response (see also existing idea)

- [[ideas/fail2ban]] already filed → nftables ban actions; pairs with layers 1–3 (its jails need the
  very log streams this idea lights up). Detection first, then banning — a ban is just a loud alert
  if nobody ever saw the attempts.

## Prerequisites & build order

1. P2 **loopback binds + Loki auth** ([[queries/security-audit]]) — non-negotiable first step;
   today's Loki is a self-serve delete button for any container
2. Layer 1 journal→Loki (Alloy settings OR revive promtail draft incl. geoip) + pipe-failure alert
3. Ruler + 4 starter LogQL rules → AM (same receiver) — ops room starts showing session cards
4. authentik login-failure metric rules (zero new infra)
5. auditd surgical rule set + `nix store verify` weekly + unit-count drift alert
6. Fail2ban (own idea), borg journald snapshots to strategos (ties [[todo/backups]])

Honest limits: catches brute force, credential stuffing, obvious persistence & tampering —
not kernel 0-days or patient attackers with your root AND your Pi. Homelab-appropriate: it makes
breaking in *noisy*, which is the entire game at this scale.

## Related

- [[todo/monitoring-alerting]] — the alerting brain this feeds
- [[ideas/fail2ban]] — active response counterpart
- [[docs/erebos/services]] — current monitoring stack description
- [[queries/security-audit]] — preconditions (binds, Loki auth)
- [[todo/backups]] — borg append-only as evidence archive
