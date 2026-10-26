---
title: Vault source of truth for blog-ideas
created: 2026-10-26
updated: 2026-10-26
type: todo
tags: [erebos, todo, housekeeping]
sources: []
---

# Vault source of truth for blog-ideas

- **Status**: Not started
- **Priority**: Medium
- **Parent**: [[todo/index]]
- **Origin**: split out as the remaining open question from the completed `todo/wiki-path` (2026-10-26)

## Description
There are two checkouts of this vault: `/home/hades/code/blog-ideas` (the user's Obsidian vault) and `/var/lib/hermes/workspace/blog-ideas` (the agent's working copy, where `WIKI_PATH` now points). They sync only through the GitHub remote — edits made locally in either copy are invisible to the other until pull/push, and simultaneous edits will eventually conflict.

## Tasks
- [ ] Decide the source of truth: single shared vault, disciplined pull-before-edit on both sides, or Obsidian Sync
- [ ] If keeping two copies: add a pull step to both workflows (agent: pull before every wiki write; user: pull before Obsidian editing)
- [ ] Alternatively relocate the Obsidian vault to `/var/lib/hermes/workspace/blog-ideas` (both users can access it) and retire the `/home/hades` copy

## Detailed notes
The agent always pulls+pushes around writes; the risk window is local Obsidian edits made while the agent has pushed newer commits. Any `git push` conflict will surface loudly (the agent's push will be rejected), so the practical failure mode is the *user's* side silently diverging if they forget to pull. Options ranked for current setup: (a) point Obsidian at the workspace copy and delete the `hades` copy — simplest; (b) Obsidian Sync for real-time convergence; (c) keep two copies with strict pull discipline.

## Related
- [[todo/index]]
- [[index]]
