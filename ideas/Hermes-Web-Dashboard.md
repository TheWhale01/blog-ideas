---
title: Hermes Web Dashboard on Erebos
created: 2026-10-22
updated: 2026-10-22
type: idea
tags: [nixos, erebos, IT, automation, documentation]
sources: [https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard]
---

# Hermes Web Dashboard on Erebos

**Status**: unimplemented (tested once locally — verified working, then stopped)

The [Hermes Web Dashboard](https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard) is a browser-based management UI for the Hermes Agent installation running on Erebos. Instead of editing YAML or using the CLI, it offers:

- **Status** — gateway state, connected platforms, active/recent sessions, auto-refresh + resource-pressure banner (memory/disk warnings).
- **Chat** — the full Hermes TUI embedded in the browser (xterm.js over PTY/WebSocket), with a session switcher rail and workspace picker.
- **Sessions** — browse, resume (▶ → `/chat?resume=<id>`), rename, export, bulk cleanup.
- **Config / API Keys / Skills / MCP / Models / Cron** — machine-level management with a profile switcher (`?profile=<name>` deep links).

## Why on Erebos

- Odin (this agent) already runs on Erebos with the Matrix gateway — the dashboard is the natural way to monitor sessions, manage skills/cron jobs, and drive an interactive chat from anywhere without SSH-ing into a terminal.
- Fits the existing homelab pattern: reverse-proxied subdomain behind Authentik, same as the rest of the stack.

## Access options

1. **SSH tunnel (zero footprint)** — run bound to loopback only (`hermes dashboard --no-open`, default port 9119), reach it with `ssh -L 9119:127.0.0.1:9119 hades@erebos` over Netbird. Good for occasional use; nothing exposed.
2. **Persistent service + protected public bind** — the production path:
   - Run as a systemd service so it survives reboots (launching it from an agent session dies with the session).
   - A **non-loopback bind always requires the dashboard's own auth gate** (`--insecure` is a no-op; fail-closed). Options: username/password, Nous Portal OAuth, or a **self-hosted OIDC provider → Authentik** ([docs: Authentication gated mode](https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard#authentication-gated-mode)).
   - Note the Chat tab keeps a WebSocket (`/api/pty`); reverse-proxy must allow WebSockets and not cap connection lifetime (idle-keepalive is handled, but hard connection timeouts cause reconnects).
   - Bind considerations: keep the server on `127.0.0.1` and let **Traefik** terminate TLS (`dashboard.thewhale.fr` or similar), with Authentik as the OIDC IdP — or the dashboard's built-in gate handling auth itself.
3. **Hermes Desktop as remote frontend** — the dashboard backend can also serve the Desktop app connecting to a remote Erebos backend.

## Security notes

- The dashboard is an **admin-equivalent surface**: it edits config, holds/manages API keys, manages skills (which execute), and its Chat tab is a real PTY running the agent. Treat it like the Authentik console: never expose without auth, prefer Netbird-only or Authentik + strong policy.
- Consider whether it should also sit behind [[ideas/Fail2ban]]-monitored logs, though Authentik-gated apps already absorb brute force.
- If exposed publicly, it complements rather than conflicts with [[ideas/Cloudflare-Tunnel-SSH]] — both remove the need for raw ports.

## Tasks

- [ ] Decide exposure model: Netbird-only vs public Authentik-gated subdomain
- [ ] Add a systemd service (or NixOS module) running `hermes dashboard --no-open` with a stable port
- [ ] Configure the dashboard auth gate (self-hosted OIDC → Authentik) before any non-loopback bind
- [ ] Wire Traefik route + WebSocket-friendly timeouts if public
- [ ] Document final setup as an implemented service page

## Related

- [[docs/erebos/services]] — Traefik/Authentik patterns this should follow
- [[ideas/Erebos-Server-TODO]] — add to backlog when implemented
- [[ideas/Cloudflare-Tunnel-SSH]] — related remote-access strategy
