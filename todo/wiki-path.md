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
- [ ] Set `WIKI_PATH=/var/lib/hermes/workspace/blog-ideas` in `/var/lib/hermes/.hermes/.env` (where the llm-wiki skill reads it)
- [ ] Clear/override the stale export found in shell session environments
- [ ] Verify in a fresh session: `echo $WIKI_PATH` → workspace copy; lint pass finds pages via the env var

## Detailed notes
Provenance: the stale value was found only in the agent's persisted terminal-session snapshot environment — not in `~/.hermes/.env`, `config.yaml`, or the gateway process environ — so it likely leaked from a shell export made during a past session. Fixing `.env` makes the correct value authoritative; the snapshot resets with a fresh terminal session.

**Divergence warning**: there are now two checkouts of this repo — `/home/hades/code/blog-ideas` (user's Obsidian vault) and the agent's workspace copy. Edits made in one are invisible to the other until synced via the git remote. Either converge both on the same remote (pull/push discipline), make the agent's workspace copy the only vault, or use Obsidian Sync — otherwise pushes from the agent will eventually conflict with local vault edits.

## Related
- [[todo/index]]
- [[index]]
