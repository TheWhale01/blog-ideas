---
title: Wiki Index
created: 2026-10-15
updated: 2026-10-15
type: index
tags: [meta, index]
---

# Wiki Index

> Content catalog. Every wiki page listed under its type with a one-line summary.
> Read this first to find relevant pages for any query.
> Last updated: 2026-11-07 | Total pages: 35

## Concepts

## Comparisons

## Ideas
- [[ideas/Deezify]] – Spotify/Deezer Jam-like feature for cross-platform music sharing
- [[ideas/Remote-Gaming-Server]] – Self-hosted cloud gaming with Sunshine/Apollo on Linux
- [[ideas/Translaitor]] – AI-powered book translator using Gemini API or local LLM (TranslateGemma)
- [[ideas/Cloudflare-Tunnel-SSH]] – Secure SSH access via Cloudflare Tunnel without opening port 22
- [[ideas/Fail2ban]] – Log-based IP banning for SSH and web services, journald/nftables integration on NixOS
- [[ideas/Intrusion-Detection]] – See attempts + know if they succeeded: journal→Loki pipeline (currently dark!), ruler alerts, authentik ground truth, auditd, NixOS drift trio, tamper-resistant evidence
- [[ideas/Chezmoi-Pontos]] – Declarative dotfiles for the pontos laptop (Fedora) with chezmoi + age, wipe-and-recover story
- [[ideas/Quickshell-Notification-Client]] – Catppuccin notification daemon as a quickconf module, built on quickshell's Quickshell.Services.Notifications
- [[ideas/CVE-Now-MCP]] – Give Odin structured CVE lookups via cve.now's free hosted MCP server (no auth; `get_vulnerability`/`list_feed`/`check_stack` incl. flake.lock) feeding [[todo/scheduled-security-audit]]
- [[ideas/Drive-Health-Monthly-Newsletter]] – Unblocked: prod is bare metal (6 ATA disks, Ryzen 5 5600) → smartctl_exporter + smartd locally → Prometheus/Grafana live view + Hermes monthly cron digest via Resend

## Todo
> Improvements/fixes for things already implemented on Erebos (new services go in Ideas).
- [[todo/index]] – Hub: all backlog items by priority
- [[todo/monitoring-alerting]] – Alerting LIVE ✅ · 9 rules deployed, webhook secret rotated + agenix-only (`85c5ebe`); open: loopback binds, Alert-link suppression
- [[todo/issues-room-readonly]] – Not started · bot-only writes in the issues room; v12 creator-PL caveat decides approach
- [[todo/backups]] – In progress · Borg chosen; full NixOS job snippet + strategos repo config + sqlite/pg_dump hooks written; weekly Sun 03:30, prune 7d/5w/12m
- [[todo/age-keys-to-strategos]] – High · agenix identity keys have no offline copy; passphrase-archive onto strategos + round-trip decrypt drill
- [[todo/terraform-workflow]] – Medium · remote state + locking, CI drift-check
- [[todo/authentik-policy-revalidation]] – Medium · username re-check at login (feasible) + forced password rotation on policy change (retroactive check impossible — hashes)
- [[todo/scheduled-security-audit]] – Medium · weekly Hermes cron job re-auditing Erebos and updating [[queries/security-audit]]

## Queries
- [[queries/security-audit]] – Living security audit of Erebos, **open findings only** (re-check 2026-11-07): P1 Immich 2.7.5 sole open P1; P2 Jellyfin/KEV-verify/binds; P3 llama.cpp drift; maintained by [[todo/scheduled-security-audit]], future CVE lookups via [[ideas/CVE-Now-MCP]]

## Documentation

### Erebos (NixOS Server)
- [[docs/erebos/index]] – Overview, repository layout, environments, and core concepts for the Erebos NixOS configuration
- [[docs/erebos/architecture]] – Big picture, layers, runtime, data flow, and security model
- [[docs/erebos/containers]] – Podman container declarations, global settings, and per-container details
- [[docs/erebos/services]] – Systemd services: Traefik, blog, Nextcloud, Matrix, Authentik, Vaultwarden, Immich, and more
- [[docs/erebos/monitoring]] – Prometheus, Grafana, Loki, and Promtail stack with Traefik integration
- [[docs/erebos/reference]] – Flake outputs, inputs, key files, Traefik routing, and NixOS options
- [[docs/erebos/secrets]] – age-encrypted secrets layout, mapping, and usage patterns
- [[docs/erebos/terraform]] – Terranix/OpenTofu workflow, providers, modules, and relationship with NixOS
- [[docs/erebos/tutorials]] – Step-by-step guides: adding services, Terraform resources, secret rotation, and the Authentik upgrade + rollback runbook (§7)

### Erebos Services (per-service docs)
- [[docs/erebos/services/opengym]] – Deployed self-hosted gym tracker: two-container Podman setup, passkey auth, Traefik routing on port 3006
- [[docs/erebos/services/hermes-dashboard]] – Agent web dashboard: loopback :9119 + Traefik + Authentik OIDC→Basic bridge

### Strategos (Raspberry Pi)
- [[docs/strategos/index]] – Pi 3B+ on Erebos' LAN: soft AP (create_ap), hardened key-only SSH, future keyring + backup receiver; 25.11-stable flake sharing nixos-modules
- [[docs/strategos/services]] – file-by-file docs of sys/ incl. the plaintext-AP-passphrase warning
- [[docs/strategos/reference]] – options table, access matrix, security notes, planned roles

---

*Original welcome page content preserved below for reference:*

This is my personal blog. It aims to contain all the ideas I have, how I solved them and what I learned. It aims to be a little personal database for a local LLM for very specific questions I could have on my dev setup / homelab environment.

Feel free to take a look around — it aims to help not only me but also others!

## Vault Structure

- **[Erebos Documentation](docs/erebos/index.md)** – NixOS server configuration and architecture.
- **[Ideas](ideas/)** – Unimplemented project ideas and drafts
