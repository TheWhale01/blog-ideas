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
> Last updated: 2026-10-22 | Total pages: 19

## Entities

## Concepts

## Comparisons

## Ideas
- [[ideas/Deezify]] – Spotify/Deezer Jam-like feature for cross-platform music sharing
- [[ideas/Remote-Gaming-Server]] – Self-hosted cloud gaming with Sunshine/Apollo on Linux
- [[ideas/Translaitor]] – AI-powered book translator using Gemini API or local LLM (TranslateGemma)
- [[ideas/Erebos-Server-TODO]] – Priority improvement list for the Erebos NixOS server
- [[ideas/Cloudflare-Tunnel-SSH]] – Secure SSH access via Cloudflare Tunnel without opening port 22
- [[ideas/Gluetun-AirVPN]] – VPN considerations for slskd with Gluetun and AirVPN options
- [[ideas/Fail2ban]] – Log-based IP banning for SSH and web services, journald/nftables integration on NixOS

## Queries

## Summaries

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
- [[docs/erebos/tutorials]] – Step-by-step guides for adding services, Terraform resources, and managing Erebos

### Erebos Services (per-service docs)
- [[docs/erebos/services/opengym]] – Deployed self-hosted gym tracker: two-container Podman setup, passkey auth, Traefik routing on port 3006

---

*Original welcome page content preserved below for reference:*

This is my personal blog. It aims to contain all the ideas I have, how I solved them and what I learned. It aims to be a little personal database for a local LLM for very specific questions I could have on my dev setup / homelab environment.

Feel free to take a look around — it aims to help not only me but also others!

## Vault Structure

- **[Erebos Documentation](docs/erebos/index.md)** – NixOS server configuration and architecture.
- **[Ideas](ideas/)** – Unimplemented project ideas and drafts
