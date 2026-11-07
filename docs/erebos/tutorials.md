---
title: Tutorials
type: tutorial
created: 2026-10-15
updated: 2026-11-07
tags: [nixos, erebos, tutorial, homelab]
sources: ["https://docs.goauthentik.io/installation/upgrade/"]
---

# Erebos — Tutorials

This directory contains step-by-step guides for common tasks on the Erebos server.

---

## 1. Add a new service to the NixOS config

1. Create a new module file under the appropriate directory:
   - `sys/services/<name>.nix` for a systemd service; or
   - `sys/containers/<name>.nix` for a Podman container.
2. Declare the service / container in Nix (see the existing modules for the pattern).
3. If the service is exposed publicly, add a Traefik dynamic config entry under `services.traefik.dynamicConfigOptions.http` (service, router, possibly a middleware for Authentik auth).
4. If it uses secrets, add the secret entry to `sys/secrets.nix` and create the `*.age` file under `secrets/`.
5. Rebuild:

```bash
sudo nixos-rebuild switch --flake .#erebos
```

## 2. Add a new Terraform/OpenTofu resource

1. Create `sys/terraform/<name>.nix`.
2. Declare resources (using the authentik / grafana / http providers as needed).
3. Add the module to `sys/terraform/default.nix` `imports`.
4. Apply:

```bash
nix run .#apply
```

## 3. Update container images

```bash
# Via the update-containers service (daily)
sudo systemctl start podman-update-containers.service   # or let it run

# Or manually:
sudo podman pull $(sudo podman images --format "{{.Repository}}:{{.Tag}}")
```

## 4. Rotate a secret

1. Regenerate the age-encrypted file: `agenix -e secrets/<env>/<name>.age` (or your preferred age key path).
2. Rebuild:

```bash
sudo nixos-rebuild switch --flake .#erebos
```

## 5. Add a new disk volume (disko)

1. Edit `sys/disko/prod.nix` (or `stage.nix`) to add a new `disk` or `lvm` device.
2. If the volume needs to be activated at boot, add an `boot.initrd.postDeviceCommands` command.
3. Rebuild with `disko` enabled.

## 6. Expose a local container publicly

1. Ensure the container is defined in `sys/containers/`.
2. Add a Traefik route in the service module:

```nix
services.traefik.dynamicConfigOptions.http = {
  services.<name>.loadBalancer.servers = [
    { url = "http://127.0.0.1:${toString vars.<port>}"; }
  ];
  routers.<name> = {
    rule = "Host(\`<name>.${vars.traefik.domain}\`)";
    tls = true;
    service = "<name>";
    entrypoints = "websecure";
  };
};
```

3. Rebuild and test.

---

## 7. Upgrade Authentik (and downgrade/rollback)

Authentik is pinned by the single `authentikVersion` variable in `sys/containers/authentik.nix`,
shared by all four images (`server`, `worker` share `server`, `ldap`, `proxy`) — the bump is always
one edit. Because the tag is pinned (not `latest`), the daily `update-containers` service never
touches Authentik. The Postgres database runs **on the host** (not in a container).
Expected SSO downtime: ~3–5 minutes — existing sessions survive briefly, new logins fail — do it
off-hours. Reference: [official upgrade docs](https://docs.goauthentik.io/installation/upgrade/).

### Upgrade

1. **Read the release notes** between the current and target version
   ([releases](https://github.com/goauthentik/authentik/releases)) — breaking changes are announced there.
2. **Back up first** (this is also your downgrade key):
   ```bash
   sudo -u postgres pg_dump authentik | gzip > /backup/authentik-pre-<oldver>.sql.gz
   sudo tar czf /backup/authentik-dirs-pre-<oldver>.tar.gz /var/lib/authentik/certs /var/lib/authentik/templates
   ```
   `media/` is large and only additive across patch releases — safe to skip.
3. **Pull the new images while the old stack still runs** (minimises downtime):
   ```bash
   sudo podman pull ghcr.io/goauthentik/server:2026.8.5 \
     ghcr.io/goauthentik/ldap:2026.8.5 ghcr.io/goauthentik/proxy:2026.8.5
   ```
4. **Stop the stack, dependents first:**
   ```bash
   sudo systemctl stop podman-authentik-ldap podman-authentik-proxy \
     podman-authentik-worker podman-authentik
   ```
5. **Run the schema migration as a one-shot** — the server container does **not** auto-migrate:
   ```bash
   sudo podman run --rm --network=podman \
     --env-file /run/secrets/authentik --env-file /run/secrets/authentik-smtp \
     -e AUTHENTIK_POSTGRESQL__HOST=host.containers.internal \
     -e AUTHENTIK_POSTGRESQL__PORT=5432 -e AUTHENTIK_POSTGRESQL__USER=authentik \
     -e AUTHENTIK_POSTGRESQL__NAME=authentik \
     ghcr.io/goauthentik/server:2026.8.5 migrate
   ```
   (If the agenix secret paths differ from `/run/secrets/…`, check `config.age.secrets.authentik.path`;
   `--network=podman` gives both container-name DNS and `host.containers.internal`.)
6. **Declare + apply the bump:** set `authentikVersion = "2026.8.5"` in `sys/containers/authentik.nix`,
   commit, `nix run .#apply`.
7. **Start and verify:**
   ```bash
   sudo systemctl start podman-authentik
   sudo systemctl start podman-authentik-worker podman-authentik-proxy podman-authentik-ldap
   ```
   Checklist: `curl -s http://localhost:9000/-/health/live/` · version shown in the admin interface ·
   one forwardAuth app sign-in works (e.g. vaultwarden) · a Jellyfin LDAP login works ·
   `journalctl -u podman-authentik-worker -b | tail` shows no silent errors.
   LDAP/proxy **outposts must match the server version** — the shared `authentikVersion` guarantees it.

If further behind than one minor: upgrade **one minor version at a time**, running step 5 between each
(`.8.5 → .9.x → …`), reading each release's notes.

### Downgrade / rollback

Authentik **does not support or document a downgrade path** — Django schema migrations are forward-only
in practice (some are reversible, some data transforms are not). The supported recovery is:

1. Stop the four `podman-authentik*` units (dependents first, as above).
2. Restore the pre-upgrade dump:
   ```bash
   gunzip -c /backup/authentik-pre-<oldver>.sql.gz | sudo -u postgres psql authentik
   ```
   (fresh DB role/schema assumed intact; if the new schema is in the way,
   `sudo -u postgres dropdb authentik && createdb -O authentik authentik` first.)
3. Restore `certs/` + `templates/` from the tarball if anything looks off (usually unnecessary).
4. Revert `authentikVersion` to the previous tag in `sys/containers/authentik.nix`, `nix run .#apply`.
   The old image is still in the local store from before the upgrade, so no pull is needed.
5. Start the stack (server first) and rerun the verification checklist above.

Last resort, **unsupported** and only for patch-level rollbacks when no backup exists: run the old
image with `migrate <app> <previous-revision>` to unwind the newest few migrations. Fragile —
prefer the restore path; treat this as a warning sign — no backup system exists yet ([[docs/erebos/architecture]] §5).

---

## Useful NixOS commands

| Task | Command |
|---|---|
| Rebuild prod | `sudo nixos-rebuild switch --flake .#erebos` |
| Rebuild stage | `sudo nixos-rebuild switch --flake .#erebos-stage` |
| Test config (no change) | `sudo nixos-rebuild test --flake .#erebos` |
| Build only | `sudo nixos-rebuild build --flake .#erebos` |
| Update inputs | `nix flake update` |
| Format | `nix fmt` |
| Validate | `nix flake check` |
| Show containers | `sudo podman ps -a` |
| Show images | `sudo podman images` |
| Pull all images | `sudo podman pull $(sudo podman images --format "{{.Repository}}:{{.Tag}}")` |

---

## Notes on the environment

- The server's timezone is `Europe/Paris`.
- The root user is the only privileged user; home-manager is managed by `hades` via `home-manager.users.hades`.
- The `hades` user is in groups: `wheel`, `libvirtd`, `video`, `input`, `uinput`, `render`, `audio`, `keys`.
- SSH: only `hades` can log in; password auth is disabled.
- The firewall allows only TCP 80 and 443; trusted interfaces are `podman0` and `wt0`.
- ~~Auto-upgrade is enabled~~ — `system.autoUpgrade` was **removed** (commit `6ef7e41`, 2026-11-04): updates are applied manually via `nix flake update` + `nix run .#apply`.
