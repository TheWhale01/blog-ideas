---
title: Secrets
tags:
  - nixos
  - erebos
  - secrets
  - agenix
  - age
---

# Erebos — Secrets

Secrets are encrypted with **age** via the `agenix` NixOS module. Each secret is a `*.age` file on disk, decrypted at build time (and made available to systemd environment files / mount points at runtime).

## Layout

```text
secrets/
├── secrets.nix            # agenix secret -> file mapping
├── shared/                # shared between prod and stage
│   ├── authentik/smtp.age
│   ├── grafana-secret.age
│   ├── gluetun.age
│   ├── hades.age
│   ├── slskd.age
│   ├── transmission.age
│   └── traefik.age
├── prod/                  # prod-only secrets
│   ├── actualbudget.age
│   ├── aurral.age
│   ├── authentik/ (authentik.age, ldap.age, proxy.age)
│   ├── grafana.age
│   ├── homepage.age
│   ├── immich.age
│   ├── matrix/ (appservice.age, mas.age, matrix.age)
│   ├── nextcloud.age
│   ├── terraform/ (authentik.age, grafana.age, jellyfin.age)
│   └── vaultwarden.age
└── stage/                 # stage-only secrets (same structure)
```

## Secrets Mapping Reference

The authoritative map lives in `sys/secrets.nix`. Key entries:

| Agens Nix path | File | Owner:Group | Mode |
|---|---|---|---|
| `age.secrets.aurral` | `secrets/{env}/aurral.age` | hades:users | 0400 |
| `age.secrets.slskd` | `secrets/shared/slskd.age` | hades:users | 0400 |
| `age.secrets.hades` | `secrets/shared/hades.age` | hades:users | 0400 |
| `age.secrets.traefik` | `secrets/shared/traefik.age` | traefik:{traefik.group} | 0440 |
| `age.secrets.gluetun` | `secrets/shared/gluetun.age` | hades:users | 0400 |
| `age.secrets.grafana-secret` | `secrets/shared/grafana-secret.age` | grafana: | 0400 |
| `age.secrets.transmission` | `secrets/shared/transmission.age` | hades:users | 0400 |
| `age.secrets.matrix-alertmanager-webhook` | `secrets/shared/matrix/alertmanager-webhook.age` | hades:users | 0400 |
| `age.secrets.erebot` | `secrets/shared/matrix/erebot.age` | hades:users | 0400 |
| `age.secrets.livekit` | `secrets/shared/matrix/livekit.age` | hades:users | 0400 |
| `age.secrets.actualbudget` | `secrets/{env}/actualbudget.age` | hades:users | 0400 |
| `age.secrets.grafana` | `secrets/{env}/grafana.age` | grafana: | 0400 |
| `age.secrets.homepage` | `secrets/{env}/homepage.age` | hades:users | 0400 |
| `age.secrets.immich` | `secrets/{env}/immich.age` | immich:{immich.group} | 0400 |
| `age.secrets.nextcloud` | `secrets/{env}/nextcloud.age` | hades:users | 0400 |
| `age.secrets.vaultwarden` | `secrets/{env}/vaultwarden.age` | hades:users | 0400 |
| `age.secrets.authentik-terraform` | `secrets/{env}/terraform/authentik.age` | hades:users | 0400 |
| `age.secrets.grafana-terraform` | `secrets/{env}/terraform/grafana.age` | hades:users | 0400 |
| `age.secrets.jellyfin-terraform` | `secrets/{env}/terraform/jellyfin.age` | hades:users | 0400 |
| `age.secrets.authentik` | `secrets/{env}/authentik/authentik.age` | hades:users | 0400 |
| `age.secrets.authentik-smtp` | `secrets/shared/authentik/smtp.age` | shared | 0400 |
| `age.secrets.authentik-ldap` | `secrets/{env}/authentik/ldap.age` | hades:users | 0400 |
| `age.secrets.authentik-proxy` | `secrets/{env}/authentik/proxy.age` | hades:users | 0400 |
| `age.secrets.matrix` | `secrets/{env}/matrix/matrix.age` | {vars.matrix.user}:{vars.matrix.group} | 0400 |
| `age.secrets.matrix-appservice` | `secrets/{env}/matrix/appservice.age` | {vars.matrix.user}:{vars.matrix.group} | 0400 |
| `age.secrets.mas` | `secrets/{env}/matrix/mas.age` | {vars.mas.user}:{vars.mas.group} | 0400 |

## How Secrets Are Used

- **Traefik**: `environmentFile = config.age.secrets.traefik.path` (daemon env).
- **Containers**: `environmentFiles = [config.age.secrets.<name>.path]`.
- **Systemd units**: `environmentFile = config.age.secrets.<name>.path`.
- **NixOS modules** reference them as `config.age.secrets.<name>.path`.

> ⚠️ `.age` files are **age-encrypted**. Never commit plaintext secrets. The age keys are provided out-of-band when building or decrypting.
