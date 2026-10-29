---
title: Authentik policy revalidation at login (username & password)
created: 2026-10-26
updated: 2026-10-26
type: todo
tags: [nixos, erebos, todo, improvement, security]
sources: []
---

# Authentik policy revalidation at login

- **Status**: Not started (researched — **feasible with workarounds**, one fundamental caveat)
- **Priority**: Medium
- **Parent**: [[todo/index]]
- **Target**: Authentik 2026.8.1 (community edition) on Erebos, configured via `sys/terraform/authentik.nix`

## Goal

When the username or password policy changes, users whose *existing* credentials violate the new rules should be intercepted at login and sent through a "change username" / "change password" step before the session is granted.

## Feasibility verdict

| Side | Verdict | Why |
|---|---|---|
| **Username** | ✅ Fully feasible | Usernames are stored in clear text — an Expression Policy can evaluate `request.user.username` against the current rules at login, and a Prompt+Write stage pair can force a rename before continuing. |
| **Password** | ⚠️ Partially feasible — retroactive check is **impossible in principle** | Authentik only stores password **hashes**; a hash cannot be re-evaluated against a new complexity/zxcvbn/HIBP policy. "Password doesn't match policy anymore" can never be detected — it can only be **approximated** by forcing a rotation: policy changed → flag users → force new password at next login (where the *new* password IS validated against the live policy). |
| **Native feature** | ❌ Not available to you yet | Issue [goauthentik/authentik#25310](https://github.com/goauthentik/authentik/issues/25310) ("Force password change at next logon") + PR [#24846](https://github.com/goauthentik/authentik/pull/24846) ("enterprise: add required actions for users") are open, milestoned 2026.11.0 — and the PR is **enterprise-only**. Your 2026.8.1 community image must use the documented community mechanism. |

## The community mechanism (documented pattern)

Authentik's official guide [Force password reset on next login](https://docs.goauthentik.io/users-sources/user/password_reset_on_login) builds exactly this with primitives you already use — [Prompt stage](https://docs.goauthentik.io/add-secure-apps/flows-stages/stages/prompt/), Expression Policies, [planner policy evaluation timing](https://docs.goauthentik.io/add-secure-apps/flows-stages/flow/planner/) ("Evaluate when stage is run"), and a Write-user stage — all declarable via the [authentik terraform provider](https://registry.terraform.io/providers/goauthentik/authentik/latest/docs/resources/stage_prompt) in `sys/terraform/authentik.nix`.

### Password leg (rotation trigger)

1. **Group-based trigger** (the guide's second variant): users in a `force-password-change` group are routed through a Prompt stage (`default-password-change-field-password` + `-repeat`, with the live [Password Policy](https://docs.goauthentik.io/customize/policies/types/password) bound as **Validation Policy**) + a Write stage that clears the flag / removes the group member afterwards — one-time interception, self-clearing.
2. **Policy-change → mass flag**: when you edit the password policy in `sys/terraform/authentik.nix`, have the same apply add all users to the group (provider `group_member` resources or a scripted `ak API` step). Terraform can't diff "policy content changed" directly — a `policy_version` marker (e.g. the group name `pwpolicy-2026-11`) that you bump manually is the honest mechanism.
3. Alternative trigger without group churn: Expression Policy comparing `request.user.password_updated` age against your rotation cadence (covers "password too old" but still not "old password violates new rules").

### Username leg (live compliance check)

1. Expression Policy `username-compliant`: regex/length/charset rules (the single definition — keep it in sync with any signup-side policy) evaluated against `request.user.username` when the flow reaches the stage binding.
2. Stage Binding #1 (policy = `not username-compliant`, evaluate-on-run): Prompt stage with a username field, its Validation Policies = the same compliance expression (rejects until valid) + uniqueness note.
3. Stage Binding #2 (runs right after): Write stage (Expression Policy setting `request.user.username = prompt_data["username"]`) — mirrors the guide's `reset_password_update` write-stage pattern.
4. Compliant users: policies fail → both stages are skipped, login proceeds unchanged.

### Gotchas

- **Username is an identity key**: apps mapping on `X-authentik-username` (forwardAuth headers — your outpost config forwards it), LDAP/Django admin, or any DB storing usernames (openGym passkey RP is ID-based, unaffected) must tolerate renames; Matrix usernames are separate. Document the rename in the flow prompt ("your username must change because …").
- Uniqueness: Prompt-stage validation can query existing users (`User.objects.filter(username=...)` in the expression) — do it, Authentik will also enforce the DB constraint but the inline error is nicer.
- Order the stage bindings **after** Identification (so `pending_user`/context.user exists) and **before** the final authorize/redirect.
- A failing policy that skips stages relies on `evaluate_on_planning = false` + `Evaluate when stage is run` — the guide and planner docs explain exactly why (at planning time the user may not be known yet).
- Keep everything in the terraform module so prod/stage stay declarative; reusing the guide's `default-password-change-*` field/prompt objects avoids reinventing them.

## Tasks

- [ ] Prototype the password leg (group-based force-change at login) in `default-authentication-flow` via the terraform provider; verify with a test user + `force-password-change` membership
- [ ] Add the compliance expression policy for usernames + Prompt/Write stage bindings; verify skip-behavior for compliant users
- [ ] Decide the "policy changed → flag users" mechanism (manual versioned group vs script in apply)
- [ ] Wire it into `sys/terraform/authentik.nix` + `nix run .#apply`, test both legs end-to-end
- [ ] Note in [[docs/erebos/services]] / hardening docs; revisit native `required actions` once 2026.11.0 ships (and whether the enterprise gate actually includes what you need)

## Related

- [[docs/erebos/containers]] — Authentik container definition
- [[todo/secrets-management]] — same motivation: policy changes must propagate, not silently rot
- [[docs/erebos/services/hermes-dashboard]] — an example of the outpost Basic-auth bridge pattern this flows through

## External references

*(all URLs curl-verified 2026-10-26)*

- **Authentik docs — Force password reset on next login** — https://docs.goauthentik.io/users-sources/user/password_reset_on_login
- **Password Policy type** — https://docs.goauthentik.io/customize/policies/types/password
- **Prompt stage** — https://docs.goauthentik.io/add-secure-apps/flows-stages/stages/prompt/
- **Flow planner / policy evaluation timing** — https://docs.goauthentik.io/add-secure-apps/flows-stages/flow/planner/
- **Native feature (open, enterprise, 2026.11.0)** — https://github.com/goauthentik/authentik/issues/25310 · https://github.com/goauthentik/authentik/pull/24846
- **authentik terraform provider** — stage_prompt: https://registry.terraform.io/providers/goauthentik/authentik/latest/docs/resources/stage_prompt · expression policy: https://registry.terraform.io/providers/goauthentik/authentik/latest/docs/resources/policy_expression
