---
title: Wiki Log
created: 2026-10-15
updated: 2026-10-15
type: log
tags: [meta, log]
---

# Wiki Log

> Chronological record of all wiki actions. Append-only.
> Format: `## [YYYY-MM-DD] action | subject`
> Actions: ingest, update, query, lint, create, archive, delete
> When this file exceeds 500 entries, rotate: rename to log-YYYY.md, start fresh.

## [2026-10-15] update | Remove stale/deprecated references
- Removed duplicate `slskd` reference in `docs/erebos/services.md`
- Updated `docs/erebos/index.md` to link to [[ideas/Erebos-Server-TODO]] instead of outdated Roadmap section
- Updated `docs/erebos/monitoring.md` to link to [[ideas/Erebos-Server-TODO#1-implement-proper-monitoring--alerting]]
- Removed legacy file note (jellyfin/default.nix, get_m3u.py) from `docs/erebos/index.md` Roadmap
- Added `reference` and `tutorial` to tag taxonomy in `SCHEMA.md`

## [2026-10-15] create | Erebos Server TODO List
  1. Implement proper monitoring & alerting
  2. Add robust backups
  3. Automate Podman image updates
  4. Centralize and secure secrets management
  5. Improve Terraform/OpenTofu workflow

## [2026-10-15] update | Wiki frontmatter standardization
- Updated frontmatter for all `ideas/*.md` and `docs/erebos/*.md` to match SCHEMA.md conventions
- Added `type`, `created`, `updated`, and `sources` fields to all pages
- Converted YAML list tags to inline array format

## [2026-10-15] update | Wiki index
- Updated `index.md` to list all 15 existing pages (4 ideas + 8 Erebos docs + 3 meta files)
- Fixed total page count from 0 to 15
- Added summaries for each page

## [2026-10-15] create | Wiki initialized
