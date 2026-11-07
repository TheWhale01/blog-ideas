---
title: Scheduled Security Audit
created: 2026-10-26
updated: 2026-10-26
type: todo
tags: [erebos, todo, improvement, automation, security]
sources: []
---

# Scheduled Security Audit (Hermes cron job)

- **Status**: Not started
- **Priority**: Medium
- **Parent**: [[todo/index]]

## Problem

[[queries/security-audit]] is a point-in-time snapshot: Erebos's exposure drifts silently (new CVEs publish against versions already running; the flake pin ages; images auto-update to new ones). A human won't re-run the full inventory + advisory cross-check by hand every week — but the audit is exactly the mechanical, cross-referencing work the agent on this box is good at.

## Proposed solution

A **Hermes Agent cron job** running on Erebos (the agent already has terminal + web access here) that, on schedule:

1. **Re-inventories live versions** the same way the first audit did: `systemctl show <unit> -p ExecStart` → Nix store paths (Traefik, llama-cpp, Synapse, Vaultwarden, n8n, Postgres, Podman, kernel `uname -r`, `os-release` pin date), container image digests (`podman images` via root or `systemctl show podman-*`), and public API probes where safe (`/status.php`, `/_matrix/client/versions`, `/System/Info/Public`, `/api/version`, `/healthz`).
2. **Diff against the previous snapshot** stored in the audit page — anything changed (up *or* down) gets called out.
3. **Cross-check advisories** per changed/notable component: web search + vendor GHSA pages for CVEs published since the last run (`GHSA`/`CISA KEV` for actively-exploited flags).
4. **Update `queries/security-audit.md`**: refresh the inventory table, re-grade P1/P2/P3, tick/untick the action list based on what's actually deployed now, append a dated entry to the **Audit log** section.
5. **Commit + push** the vault (standing wiki rule) and **report to the Matrix room** — summary only when something changed; "all clear" one-liner otherwise.

## Design notes

- Cadence: **weekly** (Sunday, after the 04:00 auto-reboot window? or Saturday to catch pre-incident state). Monthly is too slow for KEV items; daily is noise.
- Hermes' cron supports scheduled prompts delivered to a channel — see [Hermes docs: Cron jobs](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron). Job prompt should embed: audit procedure, page path (`$WIKI_PATH/queries/security-audit.md`), push rule, Matrix delivery.
- **Read-only by default**: the job audits, never patches (no rebuilds, no container restarts). Remediation stays human-approved. Optional escalation: it may *open* follow-ups in `todo/` per the wiki conventions.
- **Exposure checks must come from outside**: the 2026-10-26 run initially flagged n8n as public because testing from the host/router side is fooled by **NAT hairpinning** (a request to the WAN IP loops back internally). The job needs a true external vantage point (external probe service, or a check over Netbird from pontos, or a scheduled test from a non-LAN network) before declaring anything "internet-exposed".
- **Never trust `raw.githubusercontent.com` for "current repo truth"**: it serves cached blobs that can lag real commits by hours-to-days (cache-busting query params don't help) — caused a false "P1 not remediated" call on 2026-11-04→07 for Authentik. Use `api.github.com/repos/<owner>/<repo>/contents/<path>` (or `?ref=<branch>`) and/or `git log -S`, and always cross-check against the **live unit ExecStart / unit-script store files** on Erebos, which reveal what is actually running regardless of what the repo claims.
- Complementary tooling worth folding in later: [Trivy](https://github.com/aquasecurity/trivy) filesystem/image scanning as a deterministic pre-pass the agent then interprets against advisories; NixOS `nixpkgs` audit flags via `permittedInsecurePackages` grep in the repo.
- Sensitive-data discipline: version strings and open ports are fine in a private repo; the job must never paste secret values, tokens, or file contents into the report — names/paths/versions only.
- Failure mode: if searches fail or repo push is rejected, the job must still deliver a partial report + error note rather than silently skip.

## Tasks

- [ ] Write the audit-procedure prompt (the four phases above, with the exact commands from the 2026-10-26 run as reference)
- [ ] Create the Hermes cron job (weekly, deliver to this Matrix room, output → vault update)
- [ ] First scheduled run: validate it reproduces the manual inventory and diffs correctly
- [ ] Add Trivy pre-pass if installed via Nix (optional, later)
- [ ] Tune: what counts as "changed enough to alert" vs "log quietly"

## Related

- [[queries/security-audit]] — the page this job maintains
- [[todo/monitoring-alerting]] — overlaps; alert pipeline could later notify on top of this
- [[todo/index]] — backlog hub

## External references

*(verified 2026-10-26)*

- **Hermes Agent cron docs** — https://hermes-agent.nousresearch.com/docs/user-guide/features/cron
- **Trivy scanner** — https://github.com/aquasecurity/trivy · https://trivy.dev
- **CISA KEV catalog** (feed the job should check) — https://www.cisa.gov/known-exploited-vulnerabilities-catalog
- **GitHub Advisory Database** — https://github.com/advisories
