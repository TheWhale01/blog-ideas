---
title: Architecture
type: reference
created: 2026-10-15
updated: 2026-11-07
tags: [nixos, erebos, architecture, homelab]
sources: []
---

# Erebos — Architecture

This page explains how the pieces of the `erebos` NixOS configuration fit together.

## Big Picture

```text
                         .-------------------.
                         |   Client / Phone  |
                         `--------+----------`
                                  |
                          Netbird / WireGuard
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
- sets up systemd-boot, kernel, networking, firewall, users (`hades`), libvirtd.
  (`system.autoUpgrade` was removed in commit `6ef7e41`, 2026-11-04 — updates are applied manually
  via `nix flake update` + `nix run .#apply`.)

### 3. Modular composition

`sys/services/default.nix` imports all service modules; `sys/containers/default.nix` imports all container modules. One file per concern keeps the flake readable.

### 4. Secrets (agenix)

`sys/secrets.nix` maps each `*.age` file to a NixOS option (`config.age.secrets.<name>.path`). agenix decrypts at build time into `/run/agenix/<name>`. Modules reference secrets via these paths.

### 5. Disk layout (disko)

`sys/disko/` defines the partition tables. Prod uses a GPT disk with virtio root disk `/dev/vda`:
ESP (vfat, `/boot`) + btrfs root; `/data` is an ext4 filesystem on the pre-existing LVM volume
`/dev/mapper/whale--nas-data` (`type = "lvm"` mount entry). Since 2026-11-07 the boot runs
**systemd stage-1 initrd** (`boot.initrd.systemd.enable` default): LVM activation in the initrd is
handled automatically by `boot.initrd.services.lvm.enable` (default-on via lvm2 udev rules —
`services.lvm.enable = true`), which replaced the old scripted-era `postDeviceCommands = "lvm vgchange -ay"`
hook (it duplicated stage-1's built-in activation and was rejected by systemd stage-1; commits
`5f41664` + `034cd93`, verified by successful cold reboot with `/data` mounting). The scripted initrd
is deleted from nixpkgs in 26.11 (2026-11-30) — the old deprecation warning is gone.

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
3. Webhook edits the Markdown repo, triggers `blog-builder` webhook server, which rebuilds the site using Kiln and serves the static output via Nginx.
4. Nginx serves the static site; Traefik routes `blog.thewhale.fr` there.
5. Secrets are decrypted by agenix at build time and mounted into containers / systemd env files at runtime.

## Security model

- Public ingress is **always** behind Traefik with Let's Encrypt (Cloudflare DNS challenge).
- Most services are protected by **Authentik OIDC** (forwardAuth middleware) — everything except the blog, monitoring, and a few local-only services.
- Password auth on SSH is **disabled**; only `hades` may log in.
- Secrets are **age-encrypted**; only a user with the age keys can decrypt them.
- `mnt/terraform/` keeps the IaC layer separate from the NixOS config but driven by the same `vars.nix`.
