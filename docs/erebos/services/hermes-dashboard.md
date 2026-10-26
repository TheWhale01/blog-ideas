---
title: Hermes Dashboard
type: reference
created: 2026-10-26
updated: 2026-10-26
tags: [nixos, erebos, services, security, automation]
sources: [erebos:sys/services/hermes.nix, erebos:sys/terraform/hermes.nix, erebos:sys/vars.nix]
---

# Hermes Dashboard — Service Documentation

The **web dashboard of the Hermes Agent** (the browser management UI for "Odin"): status, embedded chat (full TUI over WebSocket/PTY), sessions, config, skills, cron — served by Erebos itself.

- **Live URL**: `https://hermes-dashboard.thewhale.fr`
- **Implemented**: 2026-10-26 (commit `27c42d9`), promoted from the former `ideas/Hermes-Web-Dashboard.md`
- **Docs**: [Hermes web dashboard](https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard)

## Architecture

Three layers, each handling one concern:

1. **Dashboard process** — `systemd.services.hermes-dashboard` runs
   `hermes dashboard --host 127.0.0.1 --port 9119 --no-open` as the `hermes-agent` service user,
   `Restart = always` / `RestartSec = 10` / `RestartPreventExitStatus = 78`.
   Loopback-only bind — nothing of the dashboard is directly reachable. Env comes from `hermes.age`.
2. **Traefik ingress** — router `hermes-dashboard.thewhale.fr` (TLS, `websecure`),
   upstream `http://127.0.0.1:9119`, `passHostHeader = false`, protected by the
   `hermes-dashboard-auth` forwardAuth middleware (`/outpost.goauthentik.io/` → `authentik-proxy`).
3. **Authentik (edge auth)** — `sys/terraform/hermes.nix` defines a proxy provider in `forward_single`
   mode; the outpost session gate + Traefik forwardAuth middleware means a visitor must authenticate
   via Authentik (policy-bound group `hermes_dashboard-users`, **members: `whale` only**) before any
   request reaches the loopback dashboard. The dashboard runs with **no auth provider of its own** —
   deliberately simplified on 2026-10-26 (commit `2de8521`); see Decision history.

## Nix Configuration

### systemd unit + Traefik (`sys/services/hermes.nix`)

```nix
systemd.services.hermes-dashboard = {
  description = "Hermes Agent Web Dashboard";
  wantedBy = [ "multi-user.target" ];
  after = [ "network.target" "hermes-agent.service" ];
  serviceConfig = {
    User = "${config.services.hermes-agent.user}";
    Group = "${config.services.hermes-agent.group}";
    EnvironmentFile = config.age.secrets.hermes.path;
    ExecStart = "/run/current-system/sw/bin/hermes dashboard --host 127.0.0.1 --port ${toString vars.hermes.dashboard.port} --no-open";
    Restart = "always";
    RestartSec = 10;
    RestartPreventExitStatus = 78;
  };
};
```

The Traefik block follows the standard outpost pattern (`services.hermes-dashboard` upstream,
`routers.hermes-dashboard` + `routers.hermes-dashboard-auth` at priority 15, `forwardAuth` to the
Authentik proxy on `vars.authentik.proxy.port`). A commented-out `dashboard.public_url` setting is
kept in `services.hermes-agent.settings` for future use.

### Variables (`sys/vars.nix`)

```nix
hermes = {
  dashboard = {
    port = 9119;
  };
};
```

### Terraform module (`sys/terraform/hermes.nix`)

Authentik resources applied via `nix run .#apply` (simplified 2026-10-26, commit `2de8521`):
- `authentik_provider_proxy.hermes_dashboard_provider` (`forward_single`, **no** Basic-auth injection)
- `authentik_application.hermes_dashboard` (slug `hermes-dashboard`, launch URL + icon for the homepage dashboard)
- `authentik_group.hermes_dashboard_users` — **members: `whale` only**
- `authentik_policy_binding` + `authentik_outpost_provider_attachment`

Secrets note: `age.secrets.hermes` is `hermes:users` mode `0440` (shared by the agent + dashboard units); the previously required `HERMES_DASHBOARD_BASIC_AUTH_*` entries were removed from the secret. The module also no longer reads/decrypts the age file via `parseEnv` — the terraform module needs no `erebos` config input anymore.

## Decision history (from the original idea)

- Chosen model: **public subdomain + Authentik-gated** (the idea also listed SSH-tunnel-only and
  Hermes-Desktop-remote options; the tunnel model was rejected as too clunky for daily use).
- The dashboard is an **admin-equivalent surface** (config editor, API keys, real PTY chat), so it
  is scoped to a single user via the Authentik policy binding; never remove the forwardAuth middleware.
- v1 (commit `27c42d9`) used an OIDC→Basic bridge: the outpost injected Basic credentials (parsed
  from `hermes.age` group attributes) to satisfy the dashboard's own auth gate. Same day it was
  **simplified to edge-only auth** (commit `2de8521`): the bridge was redundant since the dashboard
  binds to loopback and is firewalled — Authentik at the edge is the single gate.
- **Security caveat**: with no dashboard-level auth, *any* local process (or anything that gains
  loopback access on Erebos) can reach the admin UI at `:9119` directly. Accepted trade-off given
  the single-tenant host; revisit if untrusted workloads ever run on the box.

## WIKI_PATH scoping (same commit)

`2de8521` also removed the stale global `WIKI_PATH` session variable from `sys/configuration.nix`
and scoped it correctly: `systemd.services.hermes-agent.environment.WIKI_PATH =
"/var/lib/hermes/workspace/blog-ideas"`. Verified live in the gateway process environment
(2026-10-26). This closed the `todo/wiki-path` item — deleted per the todo lifecycle; the remaining
vault question carries on as [[todo/vault-source-of-truth]].

## Operational notes

- WebSocket: `/api/pty` powers the Chat tab; Traefik handles WS upgrades fine, but any hard
  connection-lifetime cap would make the chat reconnect (idle keepalives are sent every 20 s).
- The dashboard is machine-level: it manages all Hermes profiles with a profile switcher.
- Health: `Restart=always` + `RestartPreventExitStatus = 78` (clean config-error exit, no loop).
- Consider adding it to the homepage dashboard admin group (`homepage.nix` already lists Tailscale→Netbird-era tools; see [[docs/erebos/services]] Dashboard section).

## Related

- [[docs/erebos/services]] — Hermes Agent service section
- [[docs/erebos/terraform]] — IaC module list (now incl. `hermes.nix`)
- [[docs/erebos/secrets]] — `hermes.age` ownership update
- [[todo/secrets-management]] — the Basic-auth pair in group attributes should follow rotation policy
