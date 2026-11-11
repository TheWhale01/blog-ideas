---
title: Erebos Server TODO — Index
created: 2026-10-22
updated: 2026-11-11
type: todo
tags: [erebos, todo, improvement, meta]
sources: []
---

# Erebos Server TODO — Index

> Improvements, fixes, and housekeeping that change **already-implemented** configuration, code
> or services on Erebos. Anything that must be set up from scratch — new services **or new
> workflows/pipelines** — belongs in `ideas/` (see the scope rule in [[HERMES]]).
> One file per item, ordered by impact. Statuses are mirrored here; details live in each file.

## Critical
- [[todo/monitoring-alerting]] — Core DONE ✅ (12 rules live, delivery confirmed, secret rotated); log pipeline **LIVE** (Alloy→Loki 217k+ lines, alloy+loki scrapes verified); Alertmanager + Loki now loopback-bound (AM `11a4979`, Loki `f13b255`; bridge :3004 owner-declined); open: noise soak, issues room read-only, fail2ban decisions, SMART/later rules.
- [[todo/backups]] — In progress · **Borg decided**; declarative job config ready to paste (2 PG dumps + Nextcloud-SQLite quiet hook) → strategos; git-for-archives answered NO

## High
- [[todo/age-keys-to-strategos]] — Not started · agenix identity keys have NO offline copy today; passphrase-archive onto strategos + round-trip drill

## Medium
- [[todo/terraform-workflow]] — Partial · remote state + locking, CI drift-check plan, `nix run .#plan`
- [[todo/authentik-policy-revalidation]] — Not started · force username/password re-validation at login when policies change (username: fully feasible; password: force-rotation only — hashes can't be re-checked)
- [[todo/scheduled-security-audit]] — Not started · weekly Hermes cron job that re-inventories versions, cross-checks advisories, and refreshes [[queries/security-audit]]

- [[todo/fail2ban-tuning]] — New (2026-11-10) · aggressive-mode decision + ignoreIP preconditions, first web jail prototype, port-22-removal review

## Low
- [[todo/issues-room-readonly]] — Not started · issues room → send-messages level 50, @erebot PL 50; ⚠️ v12 creators can't be demoted — decide (a) creator exception vs (b) bot-created room

## History
This hub supersedes `ideas/Erebos-Server-TODO.md` (2026-10-15), which was split into the per-item files above on 2026-10-22 when the `todo/` layer was introduced. Completed & deleted items: **wiki-path** (fixed by erebos `2de8521`, verified 2026-10-26) and its follow-up **vault-source-of-truth** (decided 2026-10-26: the source of truth is the checkout at `$WIKI_PATH` — recorded in [[HERMES]]).



## Log Rotation

- Single `log.md` now holds **all** historical entries (no per-log-file renames). Future rotations just append new entries here.
- See `todo/log-rotation.md` for details.
