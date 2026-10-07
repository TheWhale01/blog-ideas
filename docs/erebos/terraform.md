---
title: Terraform
tags:
  - nixos
  - erebos
  - terraform
  - terranix
  - opentofu
  - IaC
---

# Erebos — Terraform / OpenTofu

The infrastructure-as-code layer is generated from NixOS modules via **terranix** and applied with **OpenTofu** (the open-source fork of Terraform).

## Workflow

1. The NixOS flake builds `terranixState` for `prod` / `stage` environments.
2. A flake app (`apps.<system>.apply` / `apply-stage`) generates `config.tf.json` from terranix.
3. OpenTofu is run (`tofu init` + `tofu apply`) against the generated JSON.

```bash
nix run .#apply          # prod
nix run .#apply-stage    # stage
```

## Providers

`sys/terraform/default.nix` declares the required providers:

| Provider | Source |
|---|---|
| Authentik | `goauthentik/authentik` |
| Grafana | `grafana/grafana` |
| HTTP | `hashicorp/http` |

## Terraform Modules

| Module | Purpose |
|---|---|
| `sys/terraform/default.nix` | Provider requirements + imports |
| `sys/terraform/authentik.nix` | Authentik identity provider, applications, users, groups, policies, outpost attachments |
| `sys/terraform/grafana.nix` | Grafana provisioning (admin groups, datasources, etc.) |
| `sys/terraform/nextcloud.nix` | Nextcloud resources |
| `sys/terraform/matrix.nix` | Matrix-related resources |
| `sys/terraform/actualbudget.nix` | Actual budget — Authentik app + group + policy |
| `sys/terraform/enableactual.nix` | Enable Actual — Authentik proxy provider + app + group + policy + attachment |
| `sys/terraform/immich.nix` | Immich resources |
| `sys/terraform/jellyfin.nix` | Jellyfin resources |
| `sys/terraform/lidarr.nix` | Lidarr resources |
| `sys/terraform/maintainerr.nix` | Maintainerr resources |
| `sys/terraform/openbooks.nix` | OpenBooks resources |
| `sys/terraform/prowlarr.nix` | Prowlarr resources |
| `sys/terraform/radarr.nix` | Radarr resources |
| `sys/terraform/slskd.nix` | Slskd resources |
| `sys/terraform/sonarr.nix` | Sonarr resources |
| `sys/terraform/traefik.nix` | Traefik resources |
| `sys/terraform/transmission.nix` | Transmission resources |
| `sys/terraform/vaultwarden.nix` | Vaultwarden resources |

## Notable Resources

- **Authentik** (largest module): configure the identity provider, its applications, users, groups, policies, and outpost attachments. The authentik module in particular is the mechanism by which services are wired into the identity system declaratively.
- **Grafana**: Grafana admin users and provisioning.
- Every service module creates resources that make Authentik/Grafana/HTTP providers reconcile the application state.

## Relationship with NixOS

The Terraform layer is **complementary** to the NixOS config, not a replacement. The NixOS modules define runtime services and containers; terranix/OpenTofu handles the API-level resources (identity apps, groups, policies) that the NixOS config references. This keeps the infrastructure layer versioned and reviewable in Git, while the system config stays pure Nix.
