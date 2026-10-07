---
title: Reference
tags:
  - nixos
  - erebos
  - reference
  - flake
  - outputs
---

# Erebos — Reference

## Flake Outputs

| Output | Value |
|---|---|
| `nixosConfigurations.erebos` | prod environment, `env = "prod"` |
| `nixosConfigurations.erebos-stage` | staging environment, `env = "stage"` |
| `apps.<system>.apply` | terranix → OpenTofu `tofu apply` (prod) |
| `apps.<system>.apply-stage` | terranix → OpenTofu `tofu apply` (stage) |

## Flake Inputs

| Input | Purpose |
|---|---|
| `nixpkgs` | `github:nixos/nixpkgs/release-26.05` (stable release channel) |
| `nixpkgs-unstable` | `github:nixos/nixpkgs/nixos-unstable` |
| `home-manager` | `github:nix-community/home-manager/release-26.05` |
| `agenix` | `github:ryantm/agenix` (secret decryption) |
| `disko` | `github:nix-community/disko/master` (disk layout) |
| `blog-builder` | `github:TheWhale01/blog-builder` (blog pipeline) |
| `cleanerr` | `github:TheWhale01/cleanerr?ref=test` |
| `modules` | `github:TheWhale01/nixos-modules` |
| `terranix` | `github:terranix/terranix` (NixOS → Terraform) |

## Key Configuration Files

| File | Role |
|---|---|
| `flake.nix` | Root flake: inputs, outputs, system definitions, apply apps |
| `flake.lock` | Pinned input versions |
| `sys/configuration.nix` | Main NixOS configuration |
| `sys/vars.nix` | Per-environment ports / settings table (single source of truth) |
| `sys/secrets.nix` | agenix secret → file mapping |
| `sys/home.nix` | Home Manager user `hades` |
| `sys/packages.nix` | System packages |
| `sys/hardware-configuration.nix` | Generated disk/kernel config |
| `sys/disko/prod.nix` | prod disk layout (GPT + btrfs + LVM data) |
| `sys/disko/stage.nix` | stage disk layout (GPT + btrfs) |
| `sys/terraform/default.nix` | terranix provider requirements + imports |

## Services "magic" (Traefik routing)

All public routes live in the service modules under `services/...`. Each module appends to `services.traefik.dynamicConfigOptions.http`, defining:

- `services.<name>.loadBalancer.servers` — upstream URL;
- `routers.<name>` — Traefik rule + TLS + service;
- `middlewares.<name>-auth` — forwardAuth to Authentik where applicable.

Example (from `sys/services/vaultwarden.nix`):

```nix
services.traefik.dynamicConfigOptions.http = {
  services.vaultwarden.loadBalancer.servers = [
    { url = "http://127.0.0.1:${vars.actualbudget.port}:5006"; }
  ];
  routers.vaultwarden = {
    rule = "Host(\`vaultwarden.${vars.traefik.domain}\`)";
    tls = true;
    service = "vaultwarden";
    entrypoints = "websecure";
  };
};
```

More on Traefik config: `sys/services/traefik.nix` (static + dynamic). The **base domain** is defined in `sys/vars.nix`:

```nix
traefik = {
  domain = if env == "prod" then "thewhale.fr" else "thewhale-${env}.fr";
  dns_provider = "cloudflare";
};
```

## Important NixOS Options

| Option | Default / Note |
|---|---|
| `config.services.trangers` | disabled (use Authentik OIDC) |
| `networking.firewall.allowedTCPPorts` | `[80 443]` (public) |
| `networking.firewall.trustedInterfaces` | `[podman0 tailscale0]` |
| `system.autoUpgrade` | enable + allowReboot + daily at `Sun 04:00` |
| `programs.zsh.enable` | true (default shell is zsh) |
| `virtualisation.libvirtd.enable` | true (QEMU + swtpm) |
| `nix.settings.download-buffer-size` | 500 MB |

## Secrets Reference (agenix)

For the full table, see `sys/secrets.nix`. Quick map:

| Secret | File | Owner/Group | Mode |
|---|---|---|---|
| `nextcloud` | `secrets/{env}/nextcloud.age` | hades:users | 0400 |
| `vaultwarden` | `secrets/{env}/vaultwarden.age` | hades:users | 0400 |
| `grafana` | `secrets/{env}/grafana.age` | grafana: | 0400 |
| `immich` | `secrets/{env}/immich.age` | immich: | 0400 |
| `authentik` | `secrets/{env}/authentik/authentik.age` | hades:users | 0400 |
| `authentik-smtp` | `secrets/shared/authentik/smtp.age` | shared | 0400 |
| `authentik-ldap` | `secrets/{env}/authentik/ldap.age` | hades:users | 0400 |
| `authentik-proxy` | `secrets/{env}/authentik/proxy.age` | hades:users | 0400 |
| `hades` | `secrets/shared/hades.age` | hades:users | 0400 |
| `slskd` | `secrets/shared/slskd.age` | hades:users | 0400 |
| `transmission` | `secrets/shared/transmission.age` | hades:users | 0400 |
| `gluetun` | `secrets/shared/gluetun.age` | hades:users | 0400 |
| `grafana-secret` | `secrets/shared/grafana-secret.age` | grafana: | 0400 |
| `matrix` | `secrets/{env}/matrix/matrix.age` | matrix-synapse:matrix-synapse | 0400 |
| `mas` | `secrets/{env}/matrix/mas.age` | mas:mas | 0400 |
| `matrix-appservice` | `secrets/{env}/matrix/appservice.age` | matrix-synapse:matrix-synapse | 0400 |
| `erebot` | `secrets/shared/matrix/erebot.age` | hades:users | 0400 |
| `livekit` | `secrets/shared/matrix/livekit.age` | hades:users | 0400 |
| `matrix-alertmanager-webhook` | `secrets/shared/matrix/alertmanager-webhook.age` | hades:users | 0400 |
| `homepage` | `secrets/{env}/homepage.age` | hades:users | 0400 |
| `actualbudget` | `secrets/{env}/actualbudget.age` | hades:users | 0400 |
| `aurral` | `secrets/{env}/aurral.age` | hades:users | 0400 |
| `authentik-terraform` | `secrets/{env}/terraform/authentik.age` | hades:users | 0400 |
| `grafana-terraform` | `secrets/{env}/terraform/grafana.age` | hades:users | 0400 |
| `jellyfin-terraform` | `secrets/{env}/terraform/jellyfin.age` | hades:users | 0400 |
| `traefik` | `secrets/shared/traefik.age` | traefik:traefik | 0440 |

> ⚠️ `.age` files are **age-encrypted**; never store plaintext secrets in this repo.
