---
title: Containers
tags:
  - nixos
  - erebos
  - containers
  - podman
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

**Transmission** via `gluetun`:

| Container | Image | Ports | Notes |
|---|---|---|---|
| `transmission` | `lscr.io/linuxserver/transmission:latest` | `9091` | `--network=container:gluetun`; volumes `/var/lib/transmission:/config`, `/data:/data`; secret `transmission.age`; `PUID=1000`, `PGID=100`, `TZ=Europe/Paris`, `LOG_LEVEL=debug` |
| `flood` | `docker.io/jesec/flood:latest` | `{transmission.flood.port}:3001` | `--network=container:gluetun`; `FLOOD_OPTION_auth=none`; `FLOOD_OPTION_trurl` points to `transmission:9091`; `dependsOn=transmission` |
| `transmission-port-sync` | `alpine:latest` | — | `dependsOn=transmission`; watches `/tmp/gluetun/forwarded_port` and re-registers Transmission when the VPN port changes |

Dependencies (systemd): `podman-transmission` after/requires/partOf `podman-gluetun`; `podman-flood` after/requires/partOf `podman-transmission`.

Traefik route: `transmission.thewhale.fr` → `transmission` service (auth middleware).

### `sys/containers/gluetun.nix`

| Container | Image | Notes |
|---|---|---|
| `gluetun` | `qmcgaw/gluetun:latest` | `--cap-add=NET_ADMIN`, `--cap-add=NET_RAW`, `--device=/dev/net/tun`; ports `{transmission.flood.port}:3001` (flood) + `{transmission.port}:9091` (transmission); `VPN_SERVICE_PROVIDER=protonvpn`, `VPN_TYPE=wireguard`, `SERVER_COUNTRIES=Switzerland`; `VPN_PORT_FORWARDING=on` with UP command to update transmission-remote; secrets `gluetun.age`, `transmission.age` |

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

### `sys/containers/default.nix` (bundled)

The `containers/` directory itself does not contain a slskd file; slskd is a **systemd service** defined in `sys/services/slskd.nix`.

## Service Containers (systemd)

Not in `sys/containers/` — note that slskd is a systemd service, not a container:

### `sys/services/slskd.nix`

| Option | Value |
|---|---|
| `services.slskd.enable` | true |
| `user / group` | `hades` / `users` |
| `downloads` | `/data/downloads/music/slskd/complete` |
| `incomplete` | `/data/downloads/music/slskd/incomplete` |
| `shares` | `/data/Music` |
| `web.authentication` | disabled |
| secret | `slskd.age` |
