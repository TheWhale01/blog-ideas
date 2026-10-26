---
title: Improve Terraform / OpenTofu Workflow
created: 2026-10-15
updated: 2026-10-22
type: todo
tags: [nixos, erebos, todo, improvement, infrastructure]
sources: []
---

# Improve Terraform / OpenTofu Workflow

- **Status**: Partial (terranix + OpenTofu in use; local state, no drift detection)
- **Priority**: Medium
- **Parent**: [[todo/index]]

## Description
The IaC layer is generated from NixOS modules via terranix and applied manually with the `apply`/`apply-stage` flake apps. `terraform.tfstate` lives locally on Erebos — single point of failure, no locking, no visibility into drift between applies.

## Tasks
- [ ] Move OpenTofu state to a remote backend (PostgreSQL — already running — or S3-compatible) with locking
- [ ] Add a `tofu plan -detailed-exitcode` drift-check job in GitHub Actions
- [ ] Expose `nix run .#plan` / `plan-stage` flake apps mirroring `apply`
- [ ] Document state recovery procedure (backend re-init, `tofu import`, worst-case rebuild)

## Detailed notes
State migration: take a pre-migration copy of `tfstate`, migrate with `tofu init -migrate-state`, then validate that `tofu plan` shows **zero changes** after the move.

CI: job runs `nix flake check` + terranix eval + `tofu plan -detailed-exitcode` (exit 2 = drift → fail job / comment on PR) using a **read-only** provider token stored as a repo secret.

Applies stay deliberate: CI only plans; `nix run .#apply` remains the only way to mutate.

Ordering: depends on [[todo/secrets-management]] for recipient discipline around the `secrets/{env}/terraform/` provider tokens.

Worst case is survivable: all resources are declared in `sys/terraform/*.nix`, so state loss means re-import/rebuild, not reconfiguration by hand.

## Related
- [[docs/erebos/terraform]]
- [[todo/secrets-management]]
