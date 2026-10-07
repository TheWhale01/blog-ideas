---
title: Tutorials
tags:
  - nixos
  - erebos
  - tutorial
  - homelab
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
- The firewall allows only TCP 80 and 443; trusted interfaces are `podman0` and `tailscale0`.
- Auto-upgrade is enabled (`enable = true`, `allowReboot = true`, daily at `Sun 04:00`).
