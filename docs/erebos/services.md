---
title: Services
type: reference
created: 2026-10-15
updated: 2026-11-07
tags: [nixos, erebos, services]
sources: []
---

# Erebos — Services

Each file in `sys/services/` defines one or more systemd services. Everything is declared in Nix; nothing is configured by hand after the first build.

## Central Reverse Proxy

### `sys/services/traefik.nix`

Traefik 3.7.13 as the ingress point.

- Static config: 2 entrypoints (`web` :80 → redirect to `websecure`; `websecure` :443 with timeouts 0).
- Let's Encrypt via **Cloudflare DNS challenge** (`acme.email`, `storage: acme.json`, `dnsChallenge.provider = "cloudflare"`).
- Prometheus metrics endpoint at `127.0.0.1:8200`.
- Dynamic config: Authentik **forwardAuth** middleware for the Traefik dashboard itself; `traefik-authentik-auth` middleware for apps that need OIDC.
- `environmentFile = config.age.secrets.traefik.path` (daemon env).
- Router rules expose `traefik.thewhale.fr` (dashboard + `/outpost.goauthentik.io/`).

## Media / Files

| ### `sys/services/blog.nix`

Static blog via **blog-builder** using [Kiln](https://kiln.talesign.com/) (NixOS module).

- `services.blog-builder.sites.blog-ideas` defines the site:
  - nginx port `8883`, domain `blog.thewhale.fr`;
  - `githubRepo = https://github.com/TheWhale01/blog-ideas`;
  - theme `catppuccin`, url `https://blog.thewhale.fr`, name `Whale's Blog`.
- Kiln converts the Obsidian vault to a static website with consistent theming across different color palettes.
- Traefik dynamic config wires:
  - webhook server → `blog-builder` (route `/webhook/blog-builder` on `blog.thewhale.fr`);
  - nginx site → `blog` service.

### `sys/services/nextcloud.nix`

Nextcloud + Collabora in **containers** (`sys/containers/nextcloud.nix`).

- `nextcloud` container on port `8881` (`nextcloud.port`), with `/data/nextcloud` bind mount.
- `nextcloud-office` = `collabora/code` on port `9980` (no TLS; `aliasgroup1 = https://nextcloud.thewhale.fr:443`, `--o:ssl.enable=false --o:ssl.termination=true`).
- Traefik routes: `nextcloud.thewhale.fr` and `nextcloud-office.thewhale.fr`.

### `sys/services/jellyfin/jellyfin.nix`

Jellyfin media server (via `pkgs-unstable.jellyfin`).

- `services.jellyfin.enable = true`.
- `systemd.services.get_xmltv` (daily) downloads `xmltv.zip` from `xmltvfr.fr` into `jellyfin.dataDir`.
- `systemd.services.get_m3u_file` (oneshot) converts a M3U playlist (default `mafreebox freebox.tv playlist`) into proxied RTSP URLs for Jellyfin.

### `sys/services/slskd.nix` → moved

slskd is no longer a systemd service: it was converted to a **Podman container** on 2026-10-22 and now runs behind its own VPN tunnel. See `sys/containers/slskd.nix` in [[docs/erebos/containers]] (VPN tunnels + slskd sections, incl. decision history).

## Identity & Access

### `sys/services/authentik.nix`

Authentik OIDC identity provider — the heart of the auth layer.

Containers:

- `authentik` (`ghcr.io/goauthentik/server:2026.8.5`) on `9000` (metrics `9300`).
- `authentik-worker` (same image, cmd `worker`).
- `authentik-ldap` on `3389` (cmd `ldap`).
- `authentik-proxy` on `9444` (/ `9000` internal).

Environment (declared in the module):

- `AUTHENTIK_DISABLE_STARTUP_ANALYTICS=true`
- `AUTHENTIK_AVATARS=initials`
- `AUTHENTIK_POSTGRESQL__HOST=host.containers.internal`, port `5432`
- `AUTHENTIK_EMAIL__USERNAME=resend`, host `smtp.resend.com` (port 465, SSL, no TLS).

Secrets: `authentik.age`, `authentik-smtp.age`, `authentik-ldap.age`, `authentik-proxy.age`.

### `sys/services/openssh.nix`

SSH server.

- `enable = true`, port `22`;
- `PasswordAuthentication = false`; `AllowUsers = ["hades"]`;
- `PermitRootLogin = "no"`; no X11 forwarding.

## Password Vault

### `sys/services/vaultwarden.nix`

Bitwarden-compatible self-hosted vault.

- `ROCKET_ADDRESS=127.0.0.1`, `ROCKET_PORT=8222`;
- `SSO_ENABLED=true` with OIDC (Authentik);
- `SSO_CLIENT_ID`, `SSO_AUTHORITY` → `https://authentik.thewhale.fr/application/o/vaultwarden/`;
- `SSO_ONLY=true`, `SSO_SIGNUPS_MATCH_EMAIL=true`;
- `DATABASE_URL=postgresql://%2Frun%2Fpostgresql/vaultwarden`;
- `DOMAIN=https://vaultwarden.thewhale.fr`;
- `dbBackend = "postgresql"`;
- secret: `vaultwarden.age`.

## Photo & Media

### `sys/services/immich.nix`

Self-hosted photo/video management.

- `services.immich.enable = true`, host `127.0.0.1` (was `0.0.0.0` until `b1b690e`, 2026-11-04), media location `/data/Immich`;
- `newVersionCheck.enabled = true`;
- OAuth: `autoLaunch`, `autoRegister`, `buttonText = "Login with Authentik"`, issuer `https://authentik.thewhale.fr/application/o/immich/`, scope `openid email profile`.

### `sys/services/jellyfin/get_m3u.py`

Helper script (used by the get_m3u service) to convert a freebox M3U playlist into proxied RTSP URLs for Jellyfin.

## Downloads & Torrents

### `sys/services/cleanerr.nix`

Download cleanup daemon.

- `downloadDir = "/data/downloads"`;
- `transmissionUrl = http://127.0.0.1:9091`;
- secret: `transmission.age`;
- `ignoredDirectories` (since `6ef7e41`): `/data/downloads/{aurral,lidarr,slskd}` — keeps
  media-tree downloads out of the janitor's reach.

### `sys/services/transmission` (container)

See `sys/containers/transmission.nix` — one of the more complex containers.

- `transmission` container (linuxserver/transmission) with `--network=container:gluetun-transmission`;
- `flood` container (jesec/flood) exposing a web UI, wired to `transmission`'s RPC;
- `transmission-port-sync` (alpine) polls `/tmp/gluetun/forwarded_port` and updates Transmission's port on VPN port change.

### `sys/services/gluetun` → split into two tunnels

The single shared Gluetun container no longer exists. Since 2026-10-26 there are two ProtonVPN WireGuard tunnels, defined next to their workloads: `gluetun-transmission` (in `sys/containers/transmission.nix`, Switzerland) and `gluetun-slskd` (in `sys/containers/slskd.nix`, Netherlands, `PORT_FORWARD_ONLY=on`). Details: [[docs/erebos/containers]] (VPN tunnels section).

## Streaming / Metadata

### `sys/services/seerr.nix`

Seerr media request manager (enable + proxy, behind Authentik). `services.seerr.enable = true`.

### `sys/services/prowlarr.nix`, `sys/services/radarr.nix`, `sys/services/sonarr.nix`

*arr stack indexer/manager:

- **Prowlarr** (indexer manager);
- **Radarr** (movies);
- **Sonarr** (TV shows).

Each is enabled in Nix and reverse-proxied through Traefik on its own subdomain.

### `sys/services/authentik.nix`, `sys/services/n8n.nix`, `sys/services/monitoring/`

| Service | Role |
|---|---|
| n8n | Workflow automation UI (`n8n.thewhale.fr`) — pinned to `pkgs-unstable` since commit `6ef7e41` (fresher release channel than the stable pin) |
| monitoring/ | Prometheus + Alertmanager + Grafana + Loki + Promtail + matrix-alertmanager bridge |

#### Alerting pipeline — LIVE ✅ (server-verified 2026-11-07 via event IDs in bridge journal)

Prometheus rules → Alertmanager (`:9093`) → **matrix-alertmanager 0.9.0** (`:3004`, webhook with
`?secret=` query auth) → Matrix room `!C-EIAS89vRQV7k6yUk34TTqXDro5S3Xwx_sJidUW3Lk` (ops room) as
**@erebot** — an **appservice** user (`matrix-appservice.age` registration; NOT a MAS user, MAS can
never claim it). No alert rules currently exist (deleted in `790c16a` for a from-scratch rebuild).
Config shape worth remembering:
- `matrixUser`/receiver are derived from one knob: `route.receiver = "erebot"`.
- `homeserverUrl = http://127.0.0.1:8008` — **direct to Synapse**; the public hostname's
  `mas-compat` Traefik router only matters for `/login` but direct is cleaner.
- Token: `tokenFile = age.secrets.erebot` — minted via `m.login.application_service` + as_token
  (one-time, never expires; recipe in [[todo/monitoring-alerting]]).
- Bridge **auto-joins `matrixRooms` at startup**; private rooms require an invite first (upstream README).
- Failed sends return 500 to Alertmanager (which retries); join is attempted on every send
  (upstream cache-check bug makes it unconditional — benign). Monitor the unit journal.
- ✅ Webhook secret is now **agenix-only** (`85c5ebe`): `prometheus.age` (KEY=VALUE) feeds Alertmanager
  via `environmentFile` + module envsubst; store config holds just `$MATRIX_ALERT_WEBHOOK_SECRET`;
  bridge holds the raw twin. Git history retains the old value — voided by the rotation itself.
- Message formatting (source: utils.js `formatAlert`): built-in "Alert link" (→ `generatorURL`) is a
  FALLBACK that fires only when `ALERT_LINKS` is unset. Suppress without patching via
  `systemd.services.matrix-alertmanager.serviceConfig.Environment = [ "ALERT_LINKS=Alerts:{annotations.matrixNoLink}" ];`
  — set-but-empty-resolving env var makes the fallback unreachable and the fake entry silently skips
  (single-annotation rule, utils.js:101). Real use: `Name:url|Name2:{generatorURL}` with
  `{labels.x}`/`{annotations.x}` interpolation. `services.matrix-alertmanager.mention = true`
  prefixes '@room' pings (module-supported option, default off).

#### Maintenance recipes & Matrix gotchas (learned the hard way, 2026-11-07)

- **Room version v12** (spec v1.16 "Project Hydra", Synapse default since 1.162; ours 1.164.2):
  room IDs are create-event hashes **without `:server_name`** — matrix.to/permalinks must use the
  bare ID (`https://matrix.to/#/!C-EIAS89vRQV7k6yUk34TTqXDro5S3Xwx_sJidUW3Lk`, appending the domain
  breaks resolution). **Upgrading a room yields a NEW ID** (tombstone) — if alerts vanish silently
  (journal 200s, room empty), compare live room ID in Element (Settings → Advanced) with `matrixRooms`.
  Creators hold infinite immutable power in v12 → see [[todo/issues-room-readonly]].
- **@erebot is an appservice user** (registration: `/run/agenix/matrix-appservice`, incl. `as_token`)
  — MAS can never own it (exclusive namespace). Re-mint its access token (never expires otherwise):
  ```bash
  curl -X POST http://127.0.0.1:8008/_matrix/client/v3/login \
    -H "Authorization: Bearer ***" -H 'Content-Type: application/json' \
    -d '{"type":"m.login.application_service","identifier":{"type":"server","user":"erebot"}}'
  ```
  ⚠️ **localhost:8008 only** — the public `/login` is routed to MAS by Traefik `mas-compat`, which
  rejects `m.login.application_service`. `as_token` is NOT an access token (AS endpoints only).
  Token → `agenix -e secrets/shared/matrix/erebot.age` (single line) → rebuild → invite to room if
  private (bridge auto-joins configured rooms; join attempted per send due upstream cache bug — benign).
- **MAS users (other bots)**: binary + configs from `systemctl show mas -p ExecStart` (mas-cli
  1.28.0, `--config /run/agenix/mas --config <store mas-config.yaml>`, run `sudo -u mas`);
  `manage register-user --yes <name>` + `manage issue-compatibility-token <name> <device>`;
  `.env PermissionDenied` warning is cosmetic (cwd). AS-reserved localparts fail with
  "Username not available on homeserver".
- **Bridge delivery semantics**: `receiver`→room via `MATRIX_ROOMS=<recv>/<roomId>`; webhook secret
  enforced in-app (nixpkgs patch; query `?secret=` **or Basic-auth** user `alertmanager`); Matrix send
  failure → HTTP 500 → AM retries ~5 min; successful-but-invisible → check v12 room identity first.
- **promtool** ships in the prometheus derivation's `-cli` store output:
  `ls /nix/store/*-prometheus-*-cli/bin/promtool`; unit-test rule edits via `todo/alert-rules.test.yml`.

### `sys/services/llamacpp.nix` — Local LLM (replaced Ollama, 2026-11-04)

Commit `6ef7e41` deleted `ollama.nix` and added `services.llama-cpp` from `pkgs-unstable`
(llama-cpp **0.8.0**):

- Model auto-downloaded from Hugging Face via `modelsPreset`: **`LiquidAI/LFM3-1.7B-GGUF`**, file
  `LFM3-1.7B-Q6_K.gguf` (Q6_K chosen over the ~1 GB q4 default for quality).
- Flags: `-fa on` (flash attention), `-ctk q8_0` / `-ctv q8_0` (q8 KV cache) — no explicit `-c`,
  so the server-side context is llama-server's default (verify vs Hermes' 64000 default — the
  classic local-model trap documented in the original idea).
- Binds **`127.0.0.1:8080` (loopback only)** — initially `0.0.0.0`, hardened same day in `d5340e3`
  after the audit P3; only the co-located Hermes consumes it.
- ⚠️ Upstream churn: `modelsPreset` is a **removed option** in newer nixpkgs (per-model
  `services.llama-cpp.<name>` instances). The next `nix flake update` will need this module migrated.
- **Live status 2026-11-04**: re-enabled and running on **`127.0.0.1:8080`**, router mode with both model
  entries `unloaded`; Hermes' fallback id `LiquidAI/LFM3-1.7B-GGUF:Q6_K` doesn't match the preset alias
  `LFM3-1.7B` (lazy HF fetch instead of provisioned preset) — see [[queries/security-audit]] P3.

## AI Agents

### `sys/services/hermes.nix`

The **Hermes Agent** (this assistant — "Odin") runs as a systemd service via the `services.hermes-agent` NixOS module, provided by the `hermes-agent` flake input:

- `enable = true`, `addToSystemPackages = true`.
- Model (since `d5340e3`, 2026-11-04): **primary = OpenRouter** (`qwen/qwen3.8-flash`),
  **fallback = custom endpoint** `http://127.0.0.1:8080/v1` → local **LFM3-1.7B Q6_K**
  (llama.cpp) — i.e. cloud normally serves, local inference is the no-network safety net
  (`6ef7e41` had it the other way around that same day). Keys in `secrets/shared/hermes.age`,
  `0400`, owner = hermes service user (access tightened in `6ef7e41`).
- Matrix gateway: `require_mention = true`, `allowed_users = ["@whale:matrix.thewhale.fr"]` — the agent DM room.
  Agent account `@odin:matrix.thewhale.fr`; display name **Odin** since 2026-11-07 (profile endpoint verified;
  matches the git-author identity of this assistant — see [[docs/erebos/secrets]] for `odin.age`/appservice wiring).
- Secrets: `environmentFiles = [ config.age.secrets.hermes.path ]` (`secrets/shared/hermes.age`).
- The `hades` user is member of the `hermes` group (added in `configuration.nix`, 2026-10-15) to manage the agent's state.
- Alertmanager Matrix room updated to the dedicated ops room in the same commit.
- **Web dashboard** (since 2026-10-26, commits `27c42d9` + `2de8521`): `systemd.services.hermes-dashboard` on loopback `:9119`, exposed as `hermes-dashboard.thewhale.fr` via Traefik + Authentik edge auth (forward_single outpost, whale-only group; no dashboard-side auth provider after the `2de8521` simplification). The agent unit also exports `WIKI_PATH=/var/lib/hermes/workspace/blog-ideas` in its service environment. Full details: [[docs/erebos/services/hermes-dashboard]].
- The blog pipeline renders this vault; the agent maintains it (see [[index]]).

## Dashboard

### `sys/services/homepage.nix`

Server dashboard via **homepage-dashboard** (homepage.dev).

- `environmentFiles = [config.age.secrets.homepage.path]`;
- `allowedHosts = "${vars.traefik.domain}"`;
- Homepage is served at `thewhale.fr` (root domain) via Traefik.

Layout groups:

- **Streaming** (4 cols): Jellyfin, Seerr, Immich;
- **Request Utilities** (4 cols): Radarr, Sonarr, Prowlarr, Transmission, OpenBooks;
- **Useful** (4 cols): Authentik, Matrix, Vaultwarden, Nextcloud, Blog;
- **Widgets** (4 cols): Calendar (Sonarr/Radarr) with a monthly view, first day Monday;
- **Admin Tools** (4 cols): Tailscale, Grafana, Traefik, Maintainerr.

Custom CSS forces a bold font + dark theme + blur background image (Catppuccin Mocha).

## Infrastructure (Terraform via terranix)

`sys/terraform/` maps to `terranix` and exposes OpenTofu resources. The `flake.nix` apps `apply` / `apply-stage` call `tofu apply`.

### `sys/terraform/default.nix`

Provider requirements:

- `authentik.source = "goauthentik/authentik"`;
- `grafana.source = "grafana/grafana"`;
- `http.source = "hashicorp/http"`;

### `sys/terraform/authentik.nix`

Authentik resources: identity provider, applications, users, policies, outpost attachments.

- Creates group `jellyfin-admins` (user `whale`);
- Configures proxy outpost attachment;
- Used to grant/control app access.

### `sys/terraform/grafana.nix`

- `authentik_group.grafana_admins` (admins) + policy bindings.

### `sys/terraform/actualbudget.nix`, `enableactual.nix`, `immich.nix`, `matrix.nix`, `nextcloud.nix`, `openbooks.nix`, `prowlarr.nix`, `radarr.nix`, `sonarr.nix`, `transmission.nix`, `vaultwarden.nix`, `lidarr.nix`, `slskd.nix`, `aurral.nix`, `maintainerr.nix`, `jellyfin.nix`, `hermes.nix`

Each module creates the application-level Terraform resources needed to manage the service through Authentik/Grafana/http providers.

## Misc

### `sys/services/postgresql.nix`

PostgreSQL with:

- `enable = true`, `enableTCPIP = true`;
- Auth: local peer; host 127.0.0.1 and 10.88.0.0/16 (and ::1) → scram-sha-256;
- `min_wal_size=80MB`, `max_wal_size=2GB`;
- Users: `vaultwarden`, `nextcloud`, `authentik`, `matrix-synapse`, `mas` (all `ensureDBOwnership = true`);
- Databases: `vaultwarden`, `nextcloud`, `authentik`, `matrix-synapse`, `mas`.

### `sys/services/health.nix` (or equivalent)

Some services may include health-checks; the core health endpoint is Traefik's `health` check for `websecure` entrypoint.
