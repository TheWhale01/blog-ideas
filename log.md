# Wiki Log

> Append-only chronological record of wiki actions. Format: `## [YYYY-MM-DD] action | subject`. Actions: ingest, update, query, lint, create, archive, delete. When this file exceeds 500 entries, rename to `log-YYYY.md` and start fresh.

## [2026-11-11] update | wiki sync after erebos push (HEAD `98a56d4` — Alloy log pipeline LIVE, Loki + Alertmanager loopback-bound, 12-rule alert set landed, pipeline-health alerts landed)

## [2026-11-11] delete | todo/log-rotation — removed: single `log.md` now holds all historical entries; no rotation needed

## [2026-11-11] delete | ideas/cloudflare-tunnel-ssh — removed: owner dropped the Cloudflare tunnel remote-access idea

## [2026-11-11] delete | ideas/cleanyourtv — removed: owner dropped the CleanYourTV debloat idea

## [2026-11-12] update | wiki sync after erebos push to main (HEAD `45d3721` — PR #4 homepage-dashboard internet-facing list squash-merged as `5cd2bf7`, trailing-comma syntax fix, `gh` added to hermes-agent)

- Synced `docs/erebos/services.md` Dashboard section: final group contents (Aurral→Streaming, Slskd→Request Utilities, ActualBudget/OpenGym→Useful, HermesDashboard→Admin Tools) + new "Internet-facing services on the dashboard" subsection.
- Synced `docs/erebos/services/hermes-dashboard.md`: HermesDashboard → homepage Admin Tools (PR #4).
- Synced `docs/erebos/index.md` Services Overview: added ActualBudget + Aurral rows.
- Bumped `updated` dates on touched pages to 2026-11-12.