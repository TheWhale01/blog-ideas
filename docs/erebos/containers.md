---
title: Containers
type: reference
created: 2026-10-15
updated: 2026-10-26
tags: [nixos, erebos, containers, podman]
sources: []
---

# Erebos — Containers

All containers are declared via the NixOS `virtualisation.oci-containers` module, so the entire Podman stack is declarative.

## Global Container Settings (`sys/containers/default.nix`)

- **Backend**: Podman (OCI, not Docker).
- **Storage**: overlay driver, `nodev,metacopy=on`, runroot `/run/containers/storage`, graphroot `/var/lib/containers/storage`.
- **Podman system config**:
  - `enable = true`, `dockerCompat = true`;
  - `autoPrune.enable = true` (flags: `--all --force --volumes`);
  - `defaultNetwork.settings`: `dns_enabled = true`, `ipv6_enabled = true`.
- `environment.extraInit` sets `DOCKER_HOST` from `$XDG_RUNTIME_DIR` when `DOCKER_HOST` is not set, so the host Podman CLI works for the user.
- `systemd.services.update-containers`: a daily oneshot that pulls every container image and restarts the `podman-*.service` units.

## Containers

### `sys/containers/nextcloud.nix`

| Container | Image | Ports | Notes |
|---|---|---|---|
| `nextcloud` | `nextcloud` | `{nextcloud.port}:80` → `8881` | Bind mount `/data/nextcloud:/var/www/html`; secret `nextcloud.age` |
| `nextcloud-office` | `collabora/code` | `{nextcloud.office.port}:9980` → `9980` | `capabilities.MKNOD = true`; `--o:ssl.enable=false --o:ssl.termination=true`; aliasgroup1 → `https://nextcloud.thewhale.fr:443` |

Traefik routes: `nextcloud.thewhale.fr` and `nextcloud-office.thewhale.fr`.

### `sys/containers/authentik.nix`

Authentik 2026.8.1, multi-process:

| Container | Cmd | Ports | Notes |
|---|---|---|---|
| `authentik` | `server` | `9000`, `9300` | main server |
| `authentik-worker` | `worker` | — | dependsOn authentik |
| `authentik-ldap` | `ldap` | `{authentik.ldap.port}:3389` | dependsOn authentik, `AUTHENTIK_INSECURE=true` |
| `authentik-proxy` | `proxy` | `{authentik.proxy.port}:9444` | dependsOn authentik |

Environment: `AUTHENTIK_DISABLE_STARTUP_ANALYTICS`, `AUTHENTIK_AVATARS=initials`, `AUTHENTIK_POSTGRESQL__HOST=host.containers.internal` (port 5432, user `authentik`, db `authentik`), `AUTHENTIK_EMAIL__...` → resend SMTP (port 465, SSL).

Secrets: `authentik.age`, `authentik-smtp.age`, `authentik-ldap.age`, `authentik-proxy.age`.

Route: `authentik.thewhale.fr` → `authentik` service.

### `sys/containers/transmission.nix`

**Transmission** via its own VPN tunnel `gluetun-transmission`:

| Container | Image | Ports | Notes |
|---|---|---|---|
| `transmission` | `lscr.io/linuxserver/transmission:latest` | `9091` | `--network=container:gluetun-transmission`; volumes `/var/lib/transmission:/config`, `/data:/data`; secret `transmission.age`; `PUID=1000`, `PGID=100`, `TZ=Europe/Paris`, `LOG_LEVEL=debug` |
| `flood` | `docker.io/jesec/flood:latest` | `3001` (published via `gluetun-transmission` as `{transmission.flood.port}:3001`) | `--network=container:gluetun-transmission`; `FLOOD_OPTION_auth=none`; `FLOOD_OPTION_trurl` points to `transmission:9091`; `dependsOn=transmission` |
| `transmission-port-sync` | `alpine:latest` | — | `dependsOn=transmission`; watches `/tmp/gluetun/forwarded_port` (from `gluetun-transmission` volume `/tmp/gluetun-transmission`) and re-registers Transmission when the VPN port changes |
| `gluetun-transmission` | `qmcgaw/gluetun:latest` | `{transmission.flood.port}:3001`, `{transmission.port}:9091` | ProtonVPN WireGuard, `SERVER_COUNTRIES=Switzerland`, `VPN_PORT_FORWARDING=on` (provider `protonvpn`), UP command runs `transmission-remote` with `{{PORT}}`; secrets `gluetun/transmission.age` + `transmission.age`; volumes `gluetun-transmission:/gluetun`, `/tmp/gluetun-transmission:/tmp/gluetun` |

Dependencies (systemd): `podman-transmission` after/requires/bindsTo/partOf `podman-gluetun-transmission`; `podman-flood` after/requires/bindsTo/partOf `podman-transmission`.

Traefik route: `transmission.thewhale.fr` → `transmission` service (Authentik `transmission-auth` forwardAuth middleware; `/outpost.goauthentik.io/` routed to `authentik-proxy`).

### VPN tunnels (`gluetun-transmission` + `gluetun-slskd`)

Since 2026-10-26 there are **two independent Gluetun tunnels**, each with its own ProtonVPN WireGuard device config (one WG key can only carry one tunnel) and its own agenix secret. The old single `sys/containers/gluetun.nix` module was removed; each tunnel now lives next to the workload it serves:

| Tunnel | Defined in | Country | Published ports | Secret |
|---|---|---|---|---|
| `gluetun-transmission` | `sys/containers/transmission.nix` | Switzerland | `3002:3001` (flood), `9091:9091` | `gluetun/transmission.age` |
| `gluetun-slskd` | `sys/containers/slskd.nix` | Netherlands | `{slskd.port}:5030` (slskd web UI) | `gluetun/slskd.age` |

Both: `VPN_SERVICE_PROVIDER=protonvpn`, `VPN_TYPE=wireguard`, `VPN_PORT_FORWARDING=on` with `VPN_PORT_FORWARDING_PROVIDER=protonvpn`, `--cap-add NET_ADMIN/NET_RAW`, `/dev/net/tun`, volume `/tmp/gluetun-<name>:/tmp/gluetun` for forwarded-port discovery.

`gluetun-slskd` additionally sets `PORT_FORWARD_ONLY=on` (connect only to PF-capable gateways) and exposes gluetun's HTTP control server (`127.0.0.1:8000`) so slskd can query its forwarded port natively — see below. This solved the "PCP gateway i/o timeout" that occurred when the tunnel landed on a non-PF server.

**Decision history** (2026-10-15 → 2026-10-26, formerly `ideas/Gluetun-AirVPN.md`): the goal was to escape ProtonVPN's constraints for slskd. Two options were weighed — a second Gluetun container (kept ProtonVPN) vs switching the provider to [AirVPN](https://airvpn.org/) (single credentials can drive several concurrent tunnels). Option 1 was implemented. Lessons captured on the way: one Proton WireGuard key carries exactly one tunnel (two device configs required); the initial `PCP gateway i/o timeout` on the slskd tunnel was fixed with `PORT_FORWARD_ONLY=on` + a PF-reliable country (Netherlands); slskd's native gluetun integration (`SLSKD_VPN_GLUETUN_URL`) replaced the need for a port-sync sidecar. AirVPN remains the fallback if Proton's per-device configs or PF throttling become a pain again.

### `sys/containers/slskd.nix`

slskd is now a **Podman container** (was a systemd service until 2026-10-22) sharing the `gluetun-slskd` network namespace, with **native gluetun port-forwarding integration**:

| Container | Image | Notes |
|---|---|---|
| `slskd` | `slskd/slskd:latest` | `--network=container:gluetun-slskd`; `dependsOn=gluetun-slskd`; `user 1000:100`; volumes `/data:/data`, `/var/lib/slskd:/app`; env `SLSKD_VPN=true`, `SLSKD_VPN_PORT_FORWARDING=true`, `SLSKD_VPN_GLUETUN_URL=http://127.0.0.1:8000` (auto-syncs listening port from gluetun), `SLSKD_REMOTE_FILE_MANAGEMENT=true`, dirs `/data/downloads/music/slskd/{complete,incomplete}`, shares `/data/Music`, `SLSKD_NO_AUTH=true`; secret `slskd.age` |

Traefik route: `slskd.thewhale.fr` → host port `5030` → slskd web UI, now protected by **Authentik forwardAuth** (`slskd-auth` middleware; `/outpost.goauthentik.io/` → `authentik-proxy`). `NO_AUTH` is safe because auth is enforced at the edge.

### `sys/containers/lidarr.nix`

| Container | Image | Ports | Notes |
|---|---|---|---|
| `lidarr` | `lscr.io/linuxserver/lidarr:nightly` | `{lidarr.port}:8686` | `PUID=1000`, `PGID=100`, `TZ=Europe/Paris`; volumes `/var/lib/lidarr:/config`, `/data:/data`; route via Authentik (auth middleware) |

### `sys/containers/maintainerr.nix`

| Container | Image | Ports | Notes |
|---|---|---|---|
| `maintainerr` | `ghcr.io/maintainerr/maintainerr:latest` | `{maintainerr.port}:6246` | user `1000:100`; volume `maintainerr:/opt/data` |

### `sys/containers/openbooks.nix`

| Container | Image | Ports | Notes |
|---|---|---|---|
| `openbooks` | `evanbuss/openbooks` | `{openbooks.port}:80` → `8081` | volumes `/data/Books/Books:/books`; cmd `--persist --name=openbooks`; `PGID=100`, `PUID=1000` |

### `sys/containers/actualbudget.nix`

| Container | Image | Ports | Notes |
|---|---|---|---|
| `actualbudget` | `actualbudget/actual-server:latest` | `{actualbudget.port}:5006` | OIDC: `ACTUAL_OPENID_DISCOVERY_URL`, `ACTUAL_OPENID_CLIENT_ID`, `ACTUAL_OPENID_SERVER_HOSTNAME`, `ACTUAL_USER_CREATION_MODE=login`, `ACTUAL_OPENID_ENFORCE=true`; secret `actualbudget.age` |
| `enableactual` | `2manyvcos/enable-actual` | `{enableactual.port}:3000` | `SSL_ENABLED=false`; `PUBLIC_URL=https://enableactual.thewhale.fr`; volume `enableactual:/data`; Authentik proxy middleware |

### `sys/containers/aurral.nix`

| Container | Image | Ports | Notes |
|---|---|---|---|
| `aurral` | `ghcr.io/lklynet/aurral:latest` | `{aurral.port}:3001` | `PUID=1000`, `PGID=100`; OIDC (Authentik Immich provider); volumes `/var/lib/aurral:/config`, `/data:/data`; secret `aurral.age` |

### `sys/containers/opengym.nix`

openGym runs as two containers sharing a network namespace (full details: [[docs/erebos/services/opengym]]):

| Container | Image | Ports | Notes |
|---|---|---|---|
| `opengym-api` | `ghcr.io/duartesantos8/opengym-api:latest` | `{opengym.web.port}:8080` → `3006` | API on `3000` internal; volume `/var/lib/opengym/api:/data`; passkey/WebAuthn env (`RP_ID`, `ORIGIN`, `RP_NAME`); `TRUST_PROXY=1` |
| `opengym-web` | `ghcr.io/duartesantos8/opengym-web:latest` | — | nginx on `8080`, proxies to `127.0.0.1:3000`; `dependsOn=opengym-api`; `--network=container:opengym-api` |

Traefik route: `opengym.thewhale.fr` → `opengym` service. Authentik forwardAuth middleware declared but **commented out** (service uses its own passkey auth).

### `sys/containers/default.nix` (bundled)

Imports all container modules. slskd moved here from `sys/services/` on 2026-10-22 (Podman conversion, commit `ce8fb26`).
