---
title: Remote Gaming Server
created: 2025-03-22
updated: 2026-11-12
type: idea
tags: [idea, active, infrastructure, homelab]
sources: []
---

# Remote Gaming Server

Self-host a cloud gaming service on the Erebos host ([[docs/erebos/index|Erebos]]) so
games can be streamed to Moonlight clients over the LAN — from a headless server with
no physical display.

## Status

Proposed — researched 2026-11-12; implementation not started.

## Problem statement

- Stream games from the Erebos bare-metal host (Intel Arc A380 iGPU) to a Moonlight
  client, from a headless server with no physical display / no physical access.
- The simplest option would be to get Steam Remote Play working on a headless server;
  looking at existing internet resources, it does not really seem possible.
- So the chosen path is a Sunshine-based server.

## Context and research findings (2026-11-12)

- Owner's notes: Moonlight needs a display (physical or virtual/dummy HDMI). At the time
  there was no physical access to the server, so a dummy HDMI was not an option and the
  virtual display driver seemed overly complicated. Apollo (fork of Sunshine) was found to
  support virtual displays; after build attempts it was built on Linux, though the Apollo
  `README` states availability is currently Windows-only for now.
- **2026-11-12 update**: Erebos host GPU is an **Intel Arc A380** (iGPU). Owner bought a
  **dummy HDMI plug already plugged into the GPU**, so a physical display is available
  through the iGPU — the virtual-display workaround (and therefore Apollo) is no longer
  needed. Prefer vanilla **Sunshine** + **Moonlight** (client) over Apollo.
- **2026-11-12 update**: prefer **native NixOS** (systemd) over Docker containers for the
  Sunshine server.
- Sunshine ([LizardByte/Sunshine](https://github.com/LizardByte/Sunshine)) on Linux:
  - Web UI: `https://<host>:47990`
  - Ports: HTTP 47989, HTTPS 47984, Web UI 47990, RTSP 48010, Video UDP 47998, Control UDP
    47999, Audio UDP 48000
  - Config: `~/.config/sunshine`, `apps.json` alongside; "Desktop" app hardcoded;
    per-app `prep-cmd` / `undo`; env-var overrides
- Apollo ([ClassicOldSong/Apollo](https://github.com/ClassicOldSong/Apollo)): Docker
  entrypoint `steam && sunshine`; ports `47984-47990`, `48010`, `47998-48000`; volume
  `/config`; `--device /dev/dri`; `--ipc=host`; podman `--userns=keep-id`.
- Headless stack: headless Xorg + Openbox + Picom + PipeWire; Intel Arc A380 VAAPI encode;
  automatic Moonlight resolution switching; env `SUNSHINE_CORS_ORIGIN`, `NVIDIA_GPU_ID`,
  `GAMES_DIR`.
- Web UI audio/video: encoder NVENC or VAAPI (Intel iGPU); `stream_audio`; `max_bitrate`;
  fps/resolutions.
- Moonlight: LAN client; do not expose 47990 / web UI to the internet.

## Proposed solution / implementation notes

- Architecture: vanilla Sunshine (+ Steam / Heroic) on Erebos via a NixOS systemd unit,
  Intel Arc A380 iGPU, dummy HDMI plug (physical display), Moonlight client on LAN.

### NixOS systemd unit (Sunshine AppImage)

```ini
[Unit]
Description=Sunshine Gamestream Server for Moonlight
After=graphical-session.target
Wants=graphical-session.target

[Service]
Type=simple
ExecStart=/home/<user>/sunshine.AppImage
WorkingDirectory=/home/<user>
User=<user>
Group=<user>
Restart=always

[Install]
WantedBy=graphical-session.target
```

- On Linux, Sunshine needs access to uinput (add user to `input` group, udev rules) and may
  need `sudo setcap cap_sys_admin+p $(readlink -f $(which sunshine))` for KMS use.
- Enable with `systemctl --user enable sunshine`.

- Snippets:

  ```properties
  # Sunshine config keys
  port = 47989
  max_bitrate = 0   # 0 → use the bitrate requested by Moonlight
  stream_audio = enabled
  upnp = disabled   # LAN-only by default
  ping_timeout = 10000
  min_threads = 2
  ```

- Acceptance: Moonlight discovers the server on LAN, pairs via Web UI PIN, streams a
  game with Intel Arc A380 (VAAPI) encoding.

## External references

- [Sunshine](https://github.com/LizardByte/Sunshine)
- [Apollo](https://github.com/ClassicOldSong/Apollo)
- [Sunshine DOCKER_README](https://github.com/LizardByte/Sunshine/blob/d78650572d841b5340525eb039078d74013b4f07/DOCKER_README.md)
- [Headless Sunshine + Steam Docker](https://github.com/numsu/headless-sunshine-steam-docker)
- [Sunshine configuration](https://docs.lizardbyte.dev/projects/sunshine/latest/md_docs_2configuration.html)

## Related ideas and dependencies

- [[docs/erebos/index]] · [[docs/erebos/containers]] · [[docs/erebos/services]] · [[ideas/github-pr-governance]]
