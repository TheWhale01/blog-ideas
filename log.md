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

## [2026-11-04] update | Erebos sync: commit 6ef7e41 "upgrade + running local LFM3" — idea IMPLEMENTED
- **[[ideas/Hermes-Local-LFM3]] implemented → deleted** (lifecycle rule); durable design knowledge folded
  into new `sys/services/llamacpp.nix` section in [[docs/erebos/services]]: llama-cpp 0.8.0 from
  pkgs-unstable replacing Ollama (module deleted), LFM3-1.7B **Q6_K** via HF `modelsPreset`,
  `-fa on -ctk q8_0 -ctv q8_0`; Hermes = custom provider :8080 + **fallback_providers OpenRouter**
  (qwen3.8-flash) — fallback proven live this session
- **Live observation** (from this very box): llama-cpp unit exits ~8 s after start (status 0),
  nothing on :8080 → created [[todo/llama-cpp-service-dying]] (High): journal diagnosis,
  `-c` context verification vs Hermes 64000 default, `modelsPreset` removed upstream (next flake
  update breaks eval), and P3 security note: unauth bind moved loopback → 0.0.0.0 (reachable via
  podman0/wt0 trusted interfaces)
- Also in `6ef7e41`: **`system.autoUpgrade` removed** (docs index/reference/architecture + audit
  systemic caveat updated — manual `nix flake update` cadence now; pins advanced same day,
  Authentik/Immich remediation pending version-probe), n8n → pkgs-unstable, cleanerr
  `ignoredDirectories` {aurral,lidarr,slskd}, `hermes.age` tightened 0440→0400 owner=svc user
  (secrets.md row updated)
- [[queries/security-audit]]: inventory Ollama row → llama.cpp, P3 finding added, action list +4/1
  updated, 2026-11-04 audit log entry; [[todo/scheduled-security-audit]] inventory list updated;
  [[ideas/Translaitor]] relinked (Ollama runtime gone — translategemma hosting must migrate)
- index.md: idea row removed, todo row added (30 pages); 7 pages date-bumped

## [2026-10-26] lint | Structural alignment: three consistency debts fixed
- [[HERMES]] naming rule amended: lowercase-hyphens for docs/todo/queries/raw/comparisons/concepts;
  Title-Case allowed in ideas/ (idea filenames are stable blog permalinks — renaming would break
  published URLs)
- Entity layer formalized: docs/ (docs/erebos/services/* = per-service entity pages); retired empty
  generic folders entities/ and summaries/ (removed from disk + index.md sections); _archive/
  re-scoped in Page Thresholds to rare non-git-recoverable material, supersede = delete+fold per the
  lifecycle rule
- Wikilink backfill (min-2 rule) on the three pre-convention idea pages: [[ideas/Deezify]] →
  Translaitor + docs/erebos/index; [[ideas/Remote-Gaming-Server]] → docs/erebos/index +
  Cloudflare-Tunnel-SSH; [[ideas/Translaitor]] → docs/erebos/services + Hermes-Local-LFM3
  (all link targets verified to exist; dates bumped)

## [2026-10-26] lint | Link checklist run: 89 URLs, 1 real finding
- External: 87/89 healthy. Flags: deezify.thewhale.fr → genuine 404 (offline instance; annotated
  in [[ideas/Deezify]], URL kept as historical — matches the rebuild-from-scratch plan);
  gitlab.gnome.org/GNOME/dconf → 406 is a urllib content-negotiation artifact (curl returns 200),
  link kept.
- Internal wikilinks: clean. Remaining "broken" hits are by-design: historical references in
  log.md (append-only record pointing at since-deleted pages) and the [[wikilinks]] placeholder
  example in [[HERMES]].
- Added .linkcheck-last.json to .gitignore (machine artifact of this and future runs — the
  scheduled audit job, [[todo/scheduled-security-audit]], can reuse this exact procedure)

## [2026-10-26] create | Hermes-Local-LFM3 idea
- Added [[ideas/Hermes-Local-LFM3]]: move Odin from OpenRouter (qwen3.8-flash) to a local custom
  provider running Liquid AI LFM3-1.7B (released 2026-10-27, post-trained for agentic harnesses
  incl. Hermes Agent explicitly)
- Grounded in live facts: Erebos = 12 cores / 31 GiB, Ollama already serving qwen3.5:9b +
  translategemma:4b; ollama.com/library/lfm3 confirmed with CPU quants (lfm3:1.7b-q4_ ~1 GB)
- Covers Hermes `provider: custom` config (nix snippet for sys/services/hermes.nix), the
  OLLAMA num_ctx vs context_length trap, timeouts, license (LFM Open v1.0), and the honest
  trade-off: sweet spot = scheduled/background/triage work; heavy coding stays with cloud →
  hybrid topology proposed; eval-before-switch checklist
- 6 external references, all curl-verified; index.md (total: 30 pages)

## [2026-10-26] update | Moved Scheduled-Security-Audit: ideas/ → todo/
- `git mv ideas/Scheduled-Security-Audit.md todo/scheduled-security-audit.md`; frontmatter now `type: todo`
  with Status: Not started / Priority: Medium / parent hub
- Repointed refs in queries/security-audit.md, root index.md, todo/index.md (added to Medium section)
- HERMES.md scope rule refined per user usage: todo/ = actionable backlog (improvements, fixes,
  operational/automation tasks); ideas/ = new services/projects. Page count unchanged (29)

## [2026-10-26] update | n8n audit finding corrected (P1→P3)
- Owner clarification: n8n is not internet-forwarded; built-in auth chosen deliberately (n8n SSO is paid-only)
- Root cause of my misread: NAT hairpinning — curling the WAN IP from inside the network returns 200; recorded as methodology caveat
- [[queries/security-audit]]: inventory row, findings section rewritten as P3 with residual recommendations
  (owner MFA, aggressive update cadence, LAN/VPN boundary is the wall), action list + Audit log updated
- [[ideas/Scheduled-Security-Audit]]: added "exposure checks must come from outside" design rule
- root index.md Queries line refreshed

## [2026-10-26] update | Audit page renamed + Scheduled-Security-Audit idea
- Renamed [[queries/erebos-cve-exposure-2026-10]] → [[queries/security-audit]] (git mv); reframed as
  a living document: header banner + dated Audit log section + "current snapshot" marker
- Added [[ideas/Scheduled-Security-Audit]]: weekly Hermes cron job that re-inventories versions
  (systemd/Nix-store/API probes), diffs against the stored snapshot, cross-checks vendor GHSA +
  CISA KEV, updates the audit page, pushes, and reports to Matrix — read-only by design
- External refs verified (hermes cron docs, trivy, KEV, GHSA db; vulnix links 404'd and were dropped)
- index.md: Ideas + Queries lines updated; total pages: 29

## [2026-10-26] query | Erebos CVE exposure audit (filed)
- Live inventory via systemd/Nix-store paths/service APIs (Traefik 3.7.13, Authentik 2026.8.1,
  Immich 2.7.5, n8n 2.39.6, Nextcloud 35.0.2, Jellyfin 12.2.0, Synapse 1.163.2, Ollama 0.40.3,
  Grafana 13.0.11, Vaultwarden 1.37.4, Podman 5.8.9, PG 16.15, kernel 7.3.0, pin 2026-10-27)
- Filed [[queries/erebos-cve-exposure-2026-10]] with P1/P2/P3 findings + action list:
  P1 Authentik Sep-2026 advisory batch (≤2026.8.1, fix in 2026.8.2 — MFA bypass 8.9);
  P1 Immich 2.7.5 (CVE-2026-119328 unauth ML path-injection <3.3.0, +2 more);
  P1-config n8n internet-exposed without Authentik since 5e6c83b; P2 Jellyfin ≤12.4 set;
  P2 verify richdocumentscode absent (CVE-2026-110447 CISA KEV, actively exploited);
  P2 unauthenticated monitoring ports on 0.0.0.0 (podman0 lateral surface); systemic:
  autoUpgrade doesn't advance flake pin → patch cadence gap
- Corrected stale version notes: "Traefik 2" → 3.7.13 in docs/erebos index.md + services.md
- index.md updated (total: 28 pages)

## [2026-10-26] create | Authentik policy-revalidation todo (researched)
- Added [[todo/authentik-policy-revalidation]]: intercept logins when existing creds violate changed policies
- Verdict: username leg fully feasible (Expression Policy + Prompt/Write stage bindings, official
  force-reset-on-login pattern); password leg fundamentally limited — hashed passwords can't be
  retroactively policy-checked, so it's force-rotation-on-change instead
- Native "required actions" (issue #25310 / PR #24846) is enterprise + 2026.11.0 — not for 2026.8.1 CE
- All implementation is terraform-declarable in sys/terraform/authentik.nix; 8 URLs curl-verified
- todo/index + root index.md updated (total: 27 pages)

## [2026-10-26] create | Quickshell Notification Client idea
- Added [[ideas/Quickshell-Notification-Client]]: custom notification daemon inside the quickconf
  bar repo on pontos (Hyprland/Fedora), styled with Global.qml Catppuccin Mocha
- Upstream verification against quickshell source (mirror + forge): repo moved outfoxxed/quickshell →
  quickshell-mirror/quickshell; site is quickshell.outfoxxed.me (quickshell.dev NXDOMAIN);
  module `Quickshell.Services.Notifications` already implements the whole freedesktop protocol —
  API confirmed from notification.hpp/server.hpp (NotificationServer.trackedNotifications(),
  Notification props incl. actions/hasInlineReply/dismiss()/sendInlineReply())
- Design reduced to a QML UI exercise: Repeater-bound popup window + DND gate on Global.qml;
  mako bus-name ownership check is task 1; all external URLs curl-verified per the new rule
- Updated index.md (total: 26 pages)

## [2026-10-26] update | External-references rule added; chezmoi page linked
- New standing rule in [[HERMES]] Idea Pages: every idea links official docs and/or GitHub repos
  of mentioned tools (inline for primary tool + "External references" section; URLs verified)
- [[ideas/Chezmoi-Pontos]]: added inline links + External references section (chezmoi, age, agenix,
  home-manager, Fedora packaging, flatpak, dconf) — all 13 URLs checked live with curl
- Caught during verification: chezmoi install URL is /install/ (not /user-guide/installation/);
  home-manager.org unreachable → use nix-community.github.io/home-manager

## [2026-10-26] create | Chezmoi-Pontos idea
- Added [[ideas/Chezmoi-Pontos]]: dotfile management for the laptop (pontos), which moved
  NixOS → Fedora to ease small-project development
- Covers: why chezmoi (git-native, templating, age secrets), home-manager-on-Fedora alternative
  (and why it was declined), repo structure sketch, Fedora-specific scripts (dnf/flatpak/dconf),
  wipe-and-recover drill as the acceptance test
- First idea scoped to a machine other than Erebos — noted in page
- Updated index.md (total: 25 pages)

## [2026-10-26] update | Vault source-of-truth decided; todo deleted
- User decision: the authoritative wiki checkout is wherever `WIKI_PATH` points
  (= /var/lib/hermes/workspace/blog-ideas); the /home/hades copy is not authoritative
- Recorded permanently in [[HERMES]] Domain (incl. note to point Obsidian at the $WIKI_PATH checkout)
- Deleted todo/vault-source-of-truth.md per lifecycle (last todo split from wiki-path); refs repointed
  in docs/erebos/services/hermes-dashboard, todo/index, root index.md (24 pages)

## [2026-10-26] update | Todo lifecycle: delete completed items; wiki-path closed & deleted
- HERMES.md rule changed: completed todos are DELETED (log.md + git history preserve them), no `_archive/` copies
- Verified `WIKI_PATH` fix live (gateway env = workspace path; dashboard unit active, :9119 → 200)
- Deleted todo/wiki-path.md (Done); split its still-open sub-question into new [[todo/vault-source-of-truth]] (Medium)
- Repointed refs in docs/erebos/services/hermes-dashboard, todo/index, root index.md; page count 25 (−1 +1)
- Session tracker: item 6 completed, 6b (vault source-of-truth) added

## [2026-10-26] update | Synced wiki with erebos 2de8521 (dashboard auth simplified; WIKI_PATH fixed)
- docs/erebos/services/hermes-dashboard: OIDC→Basic bridge replaced by edge-only Authentik auth
  (no dashboard-side provider; terraform module no longer parseEnv's hermes.age; group attrs removed);
  added security caveat (any local process can reach :9119), decision-history v1→v2 entry, WIKI_PATH section
- docs/erebos/services + terraform.md: matching wording updates
- todo/wiki-path: ✅ Done — WIKI_PATH removed from global sessionVariables, scoped to
  systemd.services.hermes-agent.environment (commit 2de8521); verified live in gateway env (200 on :9119,
  unit active). Follow-up kept open: decide vault source-of-truth
- todo/index + root index.md: status refreshed

## [2026-10-26] update | Hermes Dashboard implemented — idea promoted to service doc
- erebos commit `27c42d9` "added hermes dashboard": systemd unit (loopback :9119) + Traefik router
  `hermes-dashboard.thewhale.fr` + Authentik forwardAuth + new `sys/terraform/hermes.nix`
  (proxy provider with OIDC→Basic bridge via group attributes; group: whale only)
- Created [[docs/erebos/services/hermes-dashboard]] (architecture, config, decision history, ops notes)
- Deleted the implemented idea `ideas/Hermes-Web-Dashboard.md` per lifecycle rule; refs updated
- services.md (Hermes Agent section + terraform module list), docs/erebos/index services table,
  terraform.md module table, reference.md + secrets.md (hermes.age now hermes:users 0440)
- Net page count unchanged (25): −1 idea, +1 service doc

## [2026-10-26] delete | Gluetun-AirVPN idea (implemented) — lifecycle rule added
- Implemented ideas are now deleted from `ideas/` once their knowledge lives in `docs/` (rule recorded in [[HERMES]])
- [[ideas/Gluetun-AirVPN]] decision history + lessons folded into [[docs/erebos/containers]] (VPN tunnels section)
- Repointed references in [[docs/erebos/services]] and index.md; total pages: 25
- Checked remaining ideas (Deezify, Remote-Gaming-Server, Translaitor, Cloudflare-Tunnel-SSH, Fail2ban, Hermes-Web-Dashboard): all still unimplemented, they stay

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
