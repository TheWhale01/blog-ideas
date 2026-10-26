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
3. **Authentik (OIDC → Basic bridge)** — the dashboard requires its own auth provider for any
   non-loopback-facing exposure. Instead of a second OIDC gate, `sys/terraform/hermes.nix` defines a
   proxy provider in `forward_single` mode with `basic_auth_enabled = true`, and the group
   `hermes_dashboard-users` carries `username`/`password` attributes parsed from
   `HERMES_DASHBOARD_BASIC_AUTH_USERNAME` / `..._PASSWORD` in the hermes age secret. The outpost
   authenticates the user via Authentik, then injects Basic auth transparently — **one SSO login,
   both gates satisfied**.

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

Authentik resources applied via `nix run .#apply`:
- `authentik_provider_proxy.hermes_dashboard_provider` (`forward_single`, Basic auth from user attributes)
- `authentik_application.hermes_dashboard` (slug `hermes-dashboard`, launch URL + icon for the homepage dashboard)
- `authentik_group.hermes_dashboard_users` — **members: `whale` only**; attributes hold the Basic auth pair
- `authentik_policy_binding` + `authentik_outpost_provider_attachment`
- The Basic auth values are pulled from the decrypted age env file via `vars.terraform.parseEnv`
  (locals), so they never appear in the Nix store in cleartext.

Secrets note: `age.secrets.hermes` is now `hermes:users` mode `0440` (shared by the agent + dashboard units).

## Decision history (from the original idea)

- Chosen model: **public subdomain + Authentik-gated** (the idea also listed SSH-tunnel-only and
  Hermes-Desktop-remote options; the tunnel model was rejected as too clunky for daily use).
- The dashboard is an **admin-equivalent surface** (config editor, API keys, real PTY chat), so it
  is scoped to a single user via the Authentik policy binding; never remove the forwardAuth middleware.
- Dashboard auth gate: used the outpost Basic-auth bridge rather than the dashboard's own OIDC
  provider config — one IdP (Authentik), no duplicate session stores.

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
