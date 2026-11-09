---
title: GitHub PR governance for the erebos repo
created: 2026-11-09
updated: 2026-11-09
type: idea
tags: [idea, active, infrastructure, automation, security]
sources: []
---

# GitHub PR governance for the erebos repo

> Origin: owner directive in chat 2026-11-09, following an agent incident. **Filed only,
> deliberately NOT implemented** — the owner stopped the setup mid-way and asked for the
> idea to be captured here. (First filed under `todo/`, re-filed as an idea the same day
> by owner request.)

## Status

Active (proposed) — owner-driven; nothing implemented yet.

## Problem statement

Who can move the source of truth for the code deployed on [[docs/erebos/index|Erebos]]?

- The `TheWhale01/erebos` SSH key in the hermes container is linked to the owner's GitHub
  **account**, so the agent can push directly to `main` — nothing prevents it today.
- Incident 2026-11-09: agent committed the Alloy River config (see
  [[docs/erebos/monitoring]]) straight onto `main` (`c8dc1ca`) when only the config was
  requested. Unwound via lease-protected force-push; `main` = `85c5ebe` again. The change
  survives as `/var/lib/hermes/workspace/erebos-alloy.patch`.
- Deployment authority is already owner-only (rebuild/[[docs/erebos/terraform|terranix apply]]
  are human-run), but the *repo* side has no equivalent guarantee — that's the gap.
- The repo already has PR refs (`pull/1..3`) and an `erebos-stage` branch: PRs are used
  informally; `main` is unprotected.

## Proposed solution

1. **Protect `main`** (owner action, repo Settings → Rules → Branch protection rules or
   rulesets):
   - Require a pull request before merging, **required approvals = 0**. Critical nuance:
     the agent opens PRs through the owner's own OAuth token, so author = owner account,
     and GitHub forbids self-approval — an "approvals ≥ 1" rule would lock the *owner*
     out of their own PRs.
   - Block force pushes and branch deletion (default in both mechanisms; note rulesets
     cannot be bypassed by repo **admins** either — stronger than classic protection,
     and GitHub's recommended successor).
   - Optional: require conversation resolution.
2. **Agent change flow**: `feat/*` branch → PR whose body carries the summary + how it was
   verified (e.g. the Alloy config validated with `alloy fmt` + a live component boot) →
   owner reviews → agent reads review threads and pushes fixes to the same branch →
   **owner merges, owner deploys**. The agent never merges.
3. **Agent auth**: `gh` CLI via OAuth **device flow** (`repo`, `read:org`) — never a pasted
   token; credential at `~/.config/gh/hosts.yml`. Reviews consumed with
   `gh pr view --comments` + the reviews API.
4. **First PR when the flow is live**: the pending Alloy River config patch.

## Caveats & open questions

- **Trust model, honestly stated**: a device-flow token carries full `repo` scope — the
  agent could technically merge or alter protection via the API. Hard control = the
  protection/ruleset itself; soft control = the agent's standing rule (Hermes memory:
  never push/merge `main`). For a *hard* separation, the upgrade is a dedicated bot
  collaborator account (agent = author, owner = sole approver, 1 approval required).
- The 2026-11-09 setup attempt was fully rolled back: device code expired unused (no token
  was ever issued), `gh` removed from the container's nix profile, local clone reset to
  `main`, remote untouched.
- Acceptance tests once implemented:
  - `git push origin HEAD:main` → rejected by protection.
  - An agent PR with an inline review comment → agent pushes follow-up commits to the PR
    branch until threads resolve.
  - Only the owner can merge; only the owner triggers rebuilds ([[todo/terraform-workflow]]).
- Open: classic branch protection vs ruleset (ruleset recommended if only one is to be
  used); whether to squash-merge PRs onto `main` or keep merge commits.

## External references

- [About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [Managing a branch protection rule](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/managing-a-branch-protection-rule)
- [About rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
- [gh CLI](https://cli.github.com) / [cli/cli](https://github.com/cli/cli)
- [Reviews REST API](https://docs.github.com/en/rest/pulls/reviews)

## Related ideas and dependencies

[[todo/monitoring-alerting]] · [[todo/terraform-workflow]] · [[docs/erebos/monitoring]] · [[docs/erebos/secrets]]
