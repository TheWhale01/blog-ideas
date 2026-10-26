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

## [2026-10-26] update | Synced wiki with erebos commits Oct 15–26
- Read erebos repo commits: 84c3833 (hermes/Odin), 62e1e1e (opengym), ce8fb26 (slskd→podman, hermes→openrouter), c92ad7c (second gluetun)
- [[docs/erebos/containers]]: single gluetun → two tunnels (gluetun-transmission CH / gluetun-slskd NL, PF-only, per-tunnel WG key + agenix secret); new slskd-as-container section (native gluetun PF integration, Authentik fronting); transmission stack netns/secret names updated
- [[docs/erebos/services]]: slskd systemd section → pointer; gluetun section → split note; new Hermes Agent section (services.hermes-agent, OpenRouter qwen/qwen3.8-flash, Matrix allowlist, hermes.age, hades in hermes group)
- [[docs/erebos/index]]: services + containers tables, file tree (hermes.nix, slskd.nix container, no gluetun.nix), updated 2026-10-26
- [[docs/erebos/reference]] + [[docs/erebos/secrets]]: gluetun.age → gluetun/transmission.age + gluetun/slskd.age; hermes.age added
- [[ideas/Gluetun-AirVPN]]: marked Implemented via Option 1 with lessons (WG key per tunnel, PCP timeout fix, native slskd integration)
- [[todo/wiki-path]]: provenance corrected — declared in sys/configuration.nix sessionVariables (84c3833); tasks rewritten

## [2026-10-22] update | Introduced todo/ layer; split Erebos-Server-TODO
- New scope rule (recorded in [[HERMES]]): `ideas/` = new services to add; `todo/` = improvements/fixes to what's already implemented
- Created `todo/` with 7 pages: index (hub), monitoring-alerting, backups, podman-image-updates, secrets-management, terraform-workflow, wiki-path
- Each section of the former `ideas/Erebos-Server-TODO.md` became its own file (status, priority, tasks, detailed notes); hub file deleted
- Updated live references in docs/erebos/index, docs/erebos/monitoring, docs/erebos/services/opengym, ideas/Fail2ban, ideas/Hermes-Web-Dashboard, root index.md
- Tag taxonomy extended with Work: todo/improvement/fix/housekeeping; page type `todo` added
- Total pages: 26

## [2026-10-22] update | Expanded Erebos-Server-TODO with detailed notes
- Added a "Detailed notes" block to each of the 6 sections of [[ideas/Erebos-Server-TODO]]
  (implementation guidance: alert routing to Matrix, backup scope incl. /var/lib/opengym/api + pg_dumpall,
  podman auto-update labels, age recipient grouping, tofu state migration, WIKI_PATH provenance)
- Mirrors the session todo list; the wiki page is the durable source of truth

## [2026-10-22] update | Added WIKI_PATH housekeeping item to TODO list
- Appended item 6 to [[ideas/Erebos-Server-TODO]]: `WIKI_PATH` still points to the
  inaccessible `/home/hades/code/blog-ideas`; should be set to
  `/var/lib/hermes/workspace/blog-ideas` in `~/.hermes/.env`

## [2026-10-22] create | Hermes Web Dashboard idea
- Added [[ideas/Hermes-Web-Dashboard]]: browser management UI for the Hermes agent on Erebos
- Covers three access models (SSH tunnel, systemd + Authentik-gated public bind, Hermes Desktop remote backend)
- Security notes: dashboard is admin-equivalent (config, API keys, PTY chat); auth gate mandatory for non-loopback binds
- Updated index.md (total: 20 pages)
- Note: briefly launched `hermes dashboard` on loopback during research; killed and verified stopped afterwards

## [2026-10-22] create | Fail2ban idea
- Added [[ideas/Fail2ban]]: brute-force log scanning + IP bans for Erebos
- Includes NixOS specifics: journald backend (no /var/log/auth.log), nftables vs iptables actions,
  ignoreIP for podman/gluetun/wt0 ranges, X-Forwarded-For caveat for Cloudflare-fronted jails
- Cross-linked with [[ideas/Cloudflare-Tunnel-SSH]] and [[docs/erebos/services]]
- Updated index.md (total: 19 pages)

## [2026-10-22] update | Renamed SCHEMA.md to HERMES.md
- `git mv SCHEMA.md HERMES.md` — the schema file now identifies as the Hermes agent configuration for this wiki
- No content changes; historical log entries referencing SCHEMA.md left intact

## [2026-10-22] update | openGym implemented: idea promoted to service doc
- openGym is now deployed on Erebos as `sys/containers/opengym.nix`
- Moved [[ideas/openGym]] → [[docs/erebos/services/opengym]] (new `docs/erebos/services/` folder)
- Rewrote documentation to match the real Nix config:
  - Two containers: `opengym-api` (backend, port 3000) + `opengym-web` (nginx, shares network namespace)
  - Published port 3006 (vars.opengym.web.port), not 3002
  - Data dir `/var/lib/opengym/api:/data` (host), not `/data/opengym`
  - Passkey/WebAuthn auth (RP_ID/ORIGIN); Authentik forwardAuth middleware is commented out (not enforced)
  - Added env var reference table and accurate deployment commands
- Deleted ideas/openGym.md; updated index.md (Ideas minus one, new "Erebos Services" section)

## [2026-10-15] create | Gluetun and AirVPN idea
- Created [[ideas/Gluetun-AirVPN]] with VPN considerations for slskd:
  - Option 1: Second Gluetun container for Cloudflare Tunnel
  - Option 2: Switch to AirVPN to bypass ProtonVPN constraints
- Updated [[ideas/Cloudflare-Tunnel-SSH]] to remove Gluetun-specific content
- Updated index.md (total: 18 pages)

## [2026-10-15] create | Cloudflare Tunnel for SSH idea
- Added [[ideas/Cloudflare-Tunnel-SSH]] with implementation plan for exposing SSH via Cloudflare Tunnel
- Includes NixOS module, firewall rules, DNS setup, and Erebos integration steps

## [2026-10-15] update | Replace Tailscale with Netbird in architecture.md
- Updated `docs/erebos/architecture.md` to replace Tailscale with Netbird in the Big Picture diagram

## [2026-10-15] update | Replace Tailscale with Netbird references
- Updated all wiki pages to replace Tailscale with Netbird (interface: `wt0`)
- Removed `sys/services/tailscale.nix` section from `docs/erebos/services.md` (Netbird now in nixos-modules)
- Updated `docs/erebos/index.md` table and file tree to reflect Netbird via nixos-modules
- Updated `docs/erebos/tutorials.md`, `reference.md`, and `monitoring.md`

## [2026-10-15] update | Remove stale/deprecated references
- Removed duplicate `slskd` reference in `docs/erebos/services.md`
- Updated `docs/erebos/index.md` to link to [[ideas/Erebos-Server-TODO]] instead of outdated Roadmap section
- Updated `docs/erebos/index.md` to link to [[ideas/Erebos-Server-TODO]] (see "Implement Proper Monitoring & Alerting" section)
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
