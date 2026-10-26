---
title: openGym
type: reference
created: 2025-10-14
updated: 2026-10-22
tags: [nixos, erebos, services, containers]
sources: [erebos:sys/containers/opengym.nix, erebos:sys/vars.nix]
---

# openGym — Service Documentation

Self-hosted gym & body-weight tracker deployed on Erebos: plan routines, log workouts, see muscle fatigue, import from FitNotes/Strong/Hevy, and log in with passkeys. Your data, your server.

- **Homepage**: https://opengym.ch
- **GitHub**: https://github.com/DuarteSantos8/openGym
- **License**: AGPL v3
- **Live URL**: `https://opengym.thewhale.fr`

## Why openGym

Most workout apps keep your data on their servers, push subscriptions, or vanish when the company does. openGym runs on your own server, keeps your data in a folder you control, and is yours to fork. It behaves like a modern app: installable on the home screen, passkey sign-in, works offline, and syncs between devices.

No account on someone else's server, no subscription, no ads, no telemetry.

## Features

- **Planning**: Routines per weekday with a library of 5,600+ exercises, animated demos, searchable by muscle, filterable by equipment.
- **Workouts**: Supersets, warm-up sets, drop sets, rest-pause, timed exercises, cardio by time/speed, configurable rest times.
- **Tracking**: Log sets, body weight, view heatmaps, charts, and PRs.
- **Imports**: FitNotes, Strong, Hevy.
- **Auth**: Passkey login (WebAuthn).
- **PWA**: Installable on mobile/desktop.

## Architecture

openGym runs as **two Podman containers** declared in a single module, `sys/containers/opengym.nix`:

| Container | Image | Role |
|---|---|---|
| `opengym-api` | `ghcr.io/duartesantos8/opengym-api:latest` | Backend API on port `3000`; passkey/WebAuthn relying party; data in `/data` (bind-mounted from host `/var/lib/opengym/api`) |
| `opengym-web` | `ghcr.io/duartesantos8/opengym-web:latest` | Nginx frontend on port `8080`, proxying to the API at `127.0.0.1:3000` |

The two containers share a network namespace: `opengym-web` runs with `--network=container:opengym-api` and `dependsOn` the API. The **only published port** is on `opengym-api`: `${vars.opengym.web.port}:8080` (host **3006** → shared-namespace 8080 → nginx → API 3000).

**Authentication**: handled by openGym itself via **passkeys (WebAuthn)** — `RP_ID` / `ORIGIN` are set to `opengym.thewhale.fr`. The Traefik `authentik-forwardauth` middleware is **declared but commented out** in the module, so Authentik OIDC is currently *not* enforced on this service.

## Nix Configuration

### Container + routing module (`sys/containers/opengym.nix`)

```nix
{ vars, ... }:

{
  virtualisation.oci-containers.containers = {
    opengym-api = {
      image = "ghcr.io/duartesantos8/opengym-api:latest";
      volumes = [
        "/var/lib/opengym/api:/data"
      ];
      ports = [
        "${toString vars.opengym.web.port}:8080"
      ];
      environment = {
        RP_ID = "opengym.${vars.traefik.domain}";
        ORIGIN = "https://opengym.${vars.traefik.domain}";
        WEB_PORT = "${toString vars.opengym.web.port}";
        RP_NAME = "openGym";
        PORT = "3000";
        DATA_DIR = "/data";
        TRUST_PROXY = "1";
      };
    };
    opengym-web = {
      image = "ghcr.io/duartesantos8/opengym-web:latest";
      dependsOn = [ "opengym-api" ];
      environment = {
        NGINX_PORT = "8080";
        BACKEND = "127.0.0.1";
        PORT = "3000";
        RESOLVER = "1.1.1.1";
      };
      extraOptions = [ "--network=container:opengym-api" ];
    };
  };

  services.traefik.dynamicConfigOptions.http = {
    services.opengym.loadBalancer.servers = [
      { url = "http://127.0.0.1:${toString vars.opengym.web.port}"; }
    ];
    routers.opengym = {
      rule = "Host(`opengym.${vars.traefik.domain}`)";
      tls = true;
      service = "opengym";
      entrypoints = [ "websecure" ];
      # middlewares = [ "authentik-forwardauth" ];
    };
  };
}
```

### Variables (`sys/vars.nix`)

```nix
opengym = {
  web = {
    port = 3006;
  };
};
```

### Environment variables reference

| Variable | Value | Purpose |
|---|---|---|
| `RP_ID` | `opengym.thewhale.fr` | WebAuthn relying party ID (passkeys bound to this domain) |
| `ORIGIN` | `https://opengym.thewhale.fr` | WebAuthn expected origin |
| `RP_NAME` | `openGym` | Display name for passkey prompts |
| `WEB_PORT` | `3006` | Host-side published port |
| `PORT` | `3000` | API listen port (inside shared network namespace) |
| `DATA_DIR` | `/data` | Container data directory (host: `/var/lib/opengym/api`) |
| `TRUST_PROXY` | `1` | Trust Traefik reverse-proxy headers (X-Forwarded-*) |

## Deployment

```bash
# Rebuild Erebos
sudo nixos-rebuild switch --flake .#erebos

# Pull and start the containers (managed by podman-* systemd units)
sudo podman pull ghcr.io/duartesantos8/opengym-api:latest
sudo podman pull ghcr.io/duartesantos8/opengym-web:latest
sudo systemctl restart podman-opengym-api.service podman-opengym-web.service
```

## Initial Setup

- Visit `https://opengym.thewhale.fr`
- Create an admin account using a passkey
- Import workouts from FitNotes/Strong/Hevy if desired
- Configure routines and start logging

## Notes

- Data lives in `/var/lib/opengym/api` on the host — **not** under `/data` (the LVM volume). Include this path in any backup plan.
- Passkeys are bound to the `RP_ID` domain (`opengym.thewhale.fr`); changing the domain invalidates existing credentials.
- Authentik protection can be re-enabled by uncommenting the `middlewares` line in the router config.
- Container images are pulled by the daily `update-containers` service (see [[docs/erebos/containers]]).

## Related

- [[docs/erebos/index]] — Erebos overview
- [[docs/erebos/containers]] — global Podman settings and other containers
- [[docs/erebos/services]] — systemd services and Traefik wiring
- [[todo/backups]] — backup scope covers `/var/lib/opengym`
