---
title: openGym
lastmod: 2025-10-14 21:55:00
tags:
  - fitness
  - self-hosted
  - gym
  - workout-tracker
  - erebos
---

# openGym

Self-hosted gym & body-weight tracker: plan routines, log workouts, see muscle fatigue, import from FitNotes/Strong/Hevy, and log in with passkeys. Your data, your server.

- **Homepage**: https://opengym.ch
- **GitHub**: https://github.com/DuarteSantos8/openGym
- **License**: AGPL v3

## Why openGym

Most workout apps keep your data on their servers, push subscriptions, or vanish when the company does. openGym runs on your own server, keeps your data in a folder you control, and is yours to fork. It behaves like a modern app: installable on the home screen, passkey sign-in, works offline, and syncs between devices.

No account on someone else's server, no subscription, no ads, no telemetry. One `docker compose up` and it's running.

## Features

- **Planning**: Routines per weekday with a library of 5,600+ exercises, animated demos, searchable by muscle, filterable by equipment.
- **Workouts**: Supersets, warm-up sets, drop sets, rest-pause, timed exercises, cardio by time/speed, configurable rest times.
- **Tracking**: Log sets, body weight, view heatmaps, charts, and PRs.
- **Imports**: FitNotes, Strong, Hevy.
- **Auth**: Passkey login (WebAuthn).
- **PWA**: Installable on mobile/desktop.

## Erebos Integration

openGym can be deployed as a Podman container on Erebos. The recommended approach is to define a container module in `sys/containers/openGym.nix` that wires it into Traefik for HTTPS and Authentik for OIDC if desired.

### Steps

1. **Add container definition**

Create `sys/containers/openGym.nix`:

```nix
{ config, pkgs, lib, ... }:

{
  virtualisation.oci-containers.containers.openGym = {
    image = "ghcr.io/duartesantos8/opengym:latest";
    volumes = [
      "/var/lib/opengym:/app/data"
    ];
    ports = [
      "3000:3000"
    ];
    autoStart = true;
    dependsOn = [ "traefik" ];
  };
}
```

2. **Expose via Traefik**

Add a service module `sys/services/opengym.nix` to register the container with Traefik:

```nix
{ config, lib, ... }:

{
  services.opengym = {
    enable = true;
    domain = "opengym.thewhale.fr";
    port = 3000;
    traefik.enable = true;
    traefik.routers.opengym.rule = "Host(`opengym.thewhale.fr`)";
    traefik.routers.opengym.entrypoints = [ "websecure" ];
    traefik.routers.opengym.tls = true;
    traefik.routers.opengym.middlewares = [ "authentik-forwardauth" ];
  };
}
```

3. **Apply and deploy**

```bash
# Rebuild Erebos
sudo nixos-rebuild switch --flake .#erebos

# Pull and start the container
sudo podman pull ghcr.io/duartesantos8/opengym:latest
sudo systemctl restart podman-auto-update.service
```

4. **Initial setup**

- Visit https://opengym.thewhale.fr
- Create an admin account using a passkey
- Import workouts from FitNotes/Strong/Hevy if desired
- Configure routines and start logging

## Notes

- Data is stored in `/var/lib/opengym` on the host (LVM volume on prod).
- Ensure the container has write permissions to this directory.
- If using Authentik OIDC, configure the application in Authentik and update the middleware in Traefik accordingly.
- The container image is multi-arch (amd64, arm64).

## Resources

- [openGym GitHub](https://github.com/DuarteSantos8/openGym)
- [openGym Live Demo](https://opengym.ch)
- [Self-hosting guide](https://github.com/DuarteSantos8/openGym#self-hosting-guide)