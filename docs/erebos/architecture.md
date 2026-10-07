---
title: Architecture
tags:
  - nixos
  - erebos
  - architecture
  - homelab
---

# Erebos — Architecture

This page explains how the pieces of the `erebos` NixOS configuration fit together.

## Big Picture

```text
                         .-------------------.
                         |   Client / Phone  |
                         `--------+----------`
                                  |
                          Tailscale / WireGuard
                                  |
                         .--------v----------.
                         |   Erebos (server) |
                         |  NixOS (prod)       |
                         `--------+----------`
                                  |
        +------------------------+------------------------+
        |                        |                         |
  NixOS modules            terranix->OpenTofu             Podman
  (configuration.nix)      (tf.json)                     containers
        |                        |                         |
        v                        v                         v
  systemd services      cloud/IaC resources       nextcloud, trans,
  (services/, con-       + authentik, grafana,      jellyfin, etc.
   tainers/)              nextcloud, immich, ...)
        |
        v
  Secrets (agenix, age-encrypted)
```

## Layers

### 1. Flake (declarative system + IaC glue)

`flake.nix` is the root. It:

- pins every input (nixpkgs, home-manager, agenix, disko, blog-builder, cleanerr, nixos-modules, terranix);
- produces `nixosConfigurations.erebos` (env: `prod`) and `nixosConfigurations.erebos-stage` (env: `stage`);
- exposes `apps.<system>.apply` / `apply-stage` that generate `config.tf.json` via terranix and run `tofu apply`.

### 2. NixOS configuration

`sys/configuration.nix` is the main NixOS module. It:

- imports the hardware config, containers, services, secrets, disko env, and packages;
- sets up systemd-boot, kernel, networking, firewall, users (`hades`), libvirtd, auto-upgrade.

### 3. Modular composition

`sys/services/default.nix` imports all service modules; `sys/containers/default.nix` imports all container modules. One file per concern keeps the flake readable.

### 4. Secrets (agenix)

`sys/secrets.nix` maps each `*.age` file to a NixOS option (`config.age.secrets.<name>.path`). agenix decrypts at build time into `/run/agenix/<name>`. Modules reference secrets via these paths.

### 5. Disk layout (disko)

`sys/disko/` defines the partition tables. Prod uses a GPT disk with ESP + btrfs root, plus an LVM data volume (`/data`) activated by an initrd hook.

### 6. Terraform / OpenTofu (terranix)

Terranix converts NixOS modules into Terraform JSON. OpenTofu then applies cloud/API resources (authentik, grafana, etc.). The generated `config.tf.json` is the bridge between the NixOS config and the IaC provider.

## Runtime picture (prod)

```text
Internet
  |
  | 443 (TLS)
  v
Traefik (cloudflare dns challenge + authentik auth)
  |
  +---> blog.thewhale.fr (blog-builder webhook -> hugo -> nginx)
  +---> matrix.thewhale.fr (matrix-synapse + mas + livekit + lk-jwt)
  +---> authentik.thewhale.fr (identity)
  +---> nextcloud.thewhale.fr (nextcloud + collabora)
  +---> grafana.thewhale.fr (monitoring)
  +---> jellyfin.thewhale.fr (media)
  +---> vaultwarden.thewhale.fr (password vault)
  +---> immich.thewhale.fr (photos)
  +---> slskd.thewhale.fr (music)
  +---> n8n.thewhale.fr (automation)
  +---> seerr.thewhale.fr (media requests)
  +---> nextcloud-office / collab (office)
  +---> mariadb/postgresql backends
  +---> home dashboard (homepage.dev) at thewhale.fr
```

The **Podman** containers are the "data" side of the services: each container's files live on disk (`/data` LVM volume on prod, `/var/lib/...` bind mounts). The **NixOS modules** are the "control" side: they wire the containers into network, secrets, systemd, and Traefik.

## Data flow

1. You push to the `blog-ideas` GitHub repo.
2. GitHub sends a webhook to the `blog-builder` webhook server (port 8883).
3. Webhook edits the Markdown repo, triggers `blog-hugo` (Hugo build).
4. Nginx serves the static site; Traefik routes `blog.thewhale.fr` there.
5. Secrets are decrypted by agenix at build time and mounted into containers / systemd env files at runtime.

## Security model

- Public ingress is **always** behind Traefik with Let's Encrypt (Cloudflare DNS challenge).
- Most services are protected by **Authentik OIDC** (forwardAuth middleware) — everything except the blog, monitoring, and a few local-only services.
- Password auth on SSH is **disabled**; only `hades` may log in.
- Secrets are **age-encrypted**; only a user with the age keys can decrypt them.
- `mnt/terraform/` keeps the IaC layer separate from the NixOS config but driven by the same `vars.nix`.
