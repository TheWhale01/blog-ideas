---
title: Erebos Server TODO — Index
created: 2026-10-22
updated: 2026-10-22
type: todo
tags: [erebos, todo, improvement, meta]
sources: []
---

# Erebos Server TODO — Index

> Improvements, fixes, and housekeeping for services **already implemented** on Erebos.
> New services to be added belong in `ideas/` instead (see the scope rule in [[HERMES]]).
> One file per item, ordered by impact. Statuses are mirrored here; details live in each file.

## Critical
- [[todo/monitoring-alerting]] — Not started · Alertmanager + Prometheus rules + Loki queries + Grafana→Matrix routing

## High
- _(none at this level right now)_

## Medium
- [[todo/terraform-workflow]] — Partial · remote state + locking, CI drift-check plan, `nix run .#plan`
- [[todo/authentik-policy-revalidation]] — Not started · force username/password re-validation at login when policies change (username: fully feasible; password: force-rotation only — hashes can't be re-checked)
- [[todo/scheduled-security-audit]] — Not started · weekly Hermes cron job that re-inventories versions, cross-checks advisories, and refreshes [[queries/security-audit]]

## History
This hub supersedes `ideas/Erebos-Server-TODO.md` (2026-10-15), which was split into the per-item files above on 2026-10-22 when the `todo/` layer was introduced. Completed & deleted items: **wiki-path** (fixed by erebos `2de8521`, verified 2026-10-26) and its follow-up **vault-source-of-truth** (decided 2026-10-26: the source of truth is the checkout at `$WIKI_PATH` — recorded in [[HERMES]]).
