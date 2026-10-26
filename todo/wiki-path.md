---
title: "Housekeeping: fix WIKI_PATH for the blog-ideas wiki"
created: 2026-10-22
updated: 2026-10-26
type: todo
tags: [erebos, todo, housekeeping, fix]
sources: []
---

# Housekeeping: fix WIKI_PATH

- **Status**: ✅ Done (2026-10-26) — one follow-up still open (vault source-of-truth, see Tasks)
- **Priority**: Low
- **Parent**: [[todo/index]]
- **Resolution**: erebos commit `2de8521` removed the global `WIKI_PATH` from `environment.sessionVariables` in `sys/configuration.nix` and scoped it to `systemd.services.hermes-agent.environment.WIKI_PATH = "/var/lib/hermes/workspace/blog-ideas"`. Verified in the live gateway process env (2026-10-26). The fix landed via unit env rather than `~/.hermes/.env` — equivalent effect for the agent.

## Description
`WIKI_PATH` resolves to `/home/hades/code/blog-ideas`, which the `hermes` user cannot read. The working copy of the wiki (Obsidian vault + git repo) used by the agent is `/var/lib/hermes/workspace/blog-ideas/`. Any tooling that resolves `$WIKI_PATH` (llm-wiki skill defaults, cron jobs) would target the inaccessible path.

## Tasks
- [x] Fix `WIKI_PATH` in `sys/configuration.nix` (`environment.sessionVariables`) on Erebos → done via per-unit env scoping instead (cleaner)
- [x] Set `WIKI_PATH=/var/lib/hermes/workspace/blog-ideas` for the agent → achieved via the hermes-agent systemd unit
- [x] Rebuild Erebos and verify in a fresh agent session: `echo $WIKI_PATH` → workspace copy (verified via gateway process env)
- [ ] **Still open:** decide the vault-source-of-truth question (see divergence warning below)

## Detailed notes
**Provenance found (2026-10-26)**: `WIKI_PATH` is declared in Erebos's `sys/configuration.nix` — `environment.sessionVariables = rec { TERM = "xterm-256color"; EDITOR = "vim"; WIKI_PATH = "/home/hades/code/blog-ideas"; }` (added 2026-10-15, commit `84c3833`, "Added Odin user on Matrix for hermes"). It is a system-wide session variable, so the `hermes` user inherits it too — but cannot read that path. The proper fix is on the server: point the `sessionVariables` value at a path both users can access (or scope it to `hades` via home-manager), rebuild, and mirror the correct value in `/var/lib/hermes/.hermes/.env` for the agent. An earlier note here wrongly suspected a leaked shell export.

**Divergence warning**: there are now two checkouts of this repo — `/home/hades/code/blog-ideas` (user's Obsidian vault) and the agent's workspace copy. Edits made in one are invisible to the other until synced via the git remote. Either converge both on the same remote (pull/push discipline), make the agent's workspace copy the only vault, or use Obsidian Sync — otherwise pushes from the agent will eventually conflict with local vault edits.

## Related
- [[todo/index]]
- [[index]]
