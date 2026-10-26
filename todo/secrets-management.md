---
title: Centralize and Secure Secrets Management
created: 2026-10-15
updated: 2026-10-22
type: todo
tags: [nixos, erebos, todo, improvement, security]
sources: []
---

# Centralize and Secure Secrets Management

- **Status**: Partial (agenix in use, but manual rotation and broad recipients)
- **Priority**: High
- **Parent**: [[todo/index]]

## Description
Secrets are age-encrypted via agenix and split across `shared/`, `prod/`, `stage/`, but rotation is ad-hoc, most secrets list `hades` + services broadly as recipients, and there is no documented key backup.

## Tasks
- [ ] Define and run a secret rotation policy (quarterly for internet-exposed creds; on any exposure suspicion)
- [ ] Split `age` recipients into per-service groups (`age.secrets.<name>.recipientsFile`)
- [ ] Create encrypted offline backup of the master + per-service age identity keys
- [ ] Document secret ownership, recipients, last-rotated date and cadence per secret

## Detailed notes
Rotation scope: minimum for internet-exposed credentials — Authentik (incl. LDAP/proxy outpost tokens, SMTP/resend token), Nextcloud, Vaultwarden DB, Matrix secrets. Track last-rotation date per secret.

Recipients: shrink blast radius so each container/systemd unit only decrypts its own secret today.

Key backup: the master age identity (ssh-key-derived `age1...`) + per-service keys exported, encrypted to a second offline key, stored outside Erebos (password manager + passphrase-protected file). **Losing these makes every `.age` file unrecoverable** — this is the single most important item on the whole backlog.

Ownership: extend the table in [[docs/erebos/secrets]] with columns owner / recipients / last rotated / cadence.

Cross-ref: once [[todo/terraform-workflow]] moves state to a remote backend, the bucket/DB becomes secret-adjacent — apply the same recipient discipline to `secrets/{env}/terraform/`.

## Related
- [[docs/erebos/secrets]]
- [[todo/terraform-workflow]]
