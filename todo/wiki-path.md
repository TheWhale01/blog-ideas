---
title: "Housekeeping: fix WIKI_PATH for the blog-ideas wiki"
created: 2026-10-22
updated: 2026-10-22
type: todo
tags: [erebos, todo, housekeeping, fix]
sources: []
---

# Housekeeping: fix WIKI_PATH

- **Status**: Not started
- **Priority**: Low
- **Parent**: [[todo/index]]

## Description
`WIKI_PATH` resolves to `/home/hades/code/blog-ideas`, which the `hermes` user cannot read. The working copy of the wiki (Obsidian vault + git repo) used by the agent is `/var/lib/hermes/workspace/blog-ideas/`. Any tooling that resolves `$WIKI_PATH` (llm-wiki skill defaults, cron jobs) would target the inaccessible path.

## Tasks
- [ ] Fix `WIKI_PATH` in `sys/configuration.nix` (`environment.sessionVariables`) on Erebos to a path the `hermes` user can access
- [ ] Set `WIKI_PATH=/var/lib/hermes/workspace/blog-ideas` in `/var/lib/hermes/.hermes/.env` (where the llm-wiki skill reads it)
- [ ] Rebuild Erebos and verify in a fresh agent session: `echo $WIKI_PATH` → workspace copy
- [ ] Decide the vault-source-of-truth question (see divergence warning below)

## Detailed notes
**Provenance found (2026-10-26)**: `WIKI_PATH` is declared in Erebos's `sys/configuration.nix` — `environment.sessionVariables = rec { TERM = "xterm-256color"; EDITOR = "vim"; WIKI_PATH = "/home/hades/code/blog-ideas"; }` (added 2026-10-15, commit `84c3833`, "Added Odin user on Matrix for hermes"). It is a system-wide session variable, so the `hermes` user inherits it too — but cannot read that path. The proper fix is on the server: point the `sessionVariables` value at a path both users can access (or scope it to `hades` via home-manager), rebuild, and mirror the correct value in `/var/lib/hermes/.hermes/.env` for the agent. An earlier note here wrongly suspected a leaked shell export.

**Divergence warning**: there are now two checkouts of this repo — `/home/hades/code/blog-ideas` (user's Obsidian vault) and the agent's workspace copy. Edits made in one are invisible to the other until synced via the git remote. Either converge both on the same remote (pull/push discipline), make the agent's workspace copy the only vault, or use Obsidian Sync — otherwise pushes from the agent will eventually conflict with local vault edits.

## Related
- [[todo/index]]
- [[index]]
