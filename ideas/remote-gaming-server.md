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
  virtual display driver seemed overly complicated; a physical dummy HDMI plug has since
  been bought and plugged into the GPU, removing the need for a virtual display.
- **2026-11-12 update**: Erebos host GPU is an **Intel Arc A380** (iGPU). Owner bought a
  **dummy HDMI plug already plugged into the GPU**, so a physical display is available
  through the iGPU — the virtual-display workaround is no longer needed.
- **2026-11-12 update**: prefer the **NixOS native module** (`services.sunshine`) over
  Docker / hand-written systemd for the Sunshine server.
- Sunshine ([LizardByte/Sunshine](https://github.com/LizardByte/Sunshine)) on Linux:
  - Web UI: `https://<host>:47990`
  - Ports: HTTP 47989, HTTPS 47984, Web UI 47990, RTSP 48010, Video UDP 47998, Control UDP
    47999, Audio UDP 48000
  - Config: `~/.config/sunshine`, `apps.json` alongside; "Desktop" app hardcoded;
    per-app `prep-cmd` / `undo`; env-var overrides
- Headless stack: headless Xorg + Openbox + Picom + PipeWire; Intel Arc A380 VAAPI encode;
  automatic Moonlight resolution switching; env `SUNSHINE_CORS_ORIGIN`, `NVIDIA_GPU_ID`,
  `GAMES_DIR`.
- Web UI audio/video: encoder NVENC or VAAPI (Intel iGPU); `stream_audio`; `max_bitrate`;
  fps/resolutions.
- Moonlight: LAN client; do not expose 47990 / web UI to the internet.

## Proposed solution / implementation notes

- Architecture: vanilla Sunshine (+ Steam / Heroic) via the **NixOS native module**
  (`services.sunshine`), Intel Arc A380 iGPU, dummy HDMI plug (physical display), Moonlight
  client on LAN.

```nix
services.sunshine = {
  enable = true;
  openFirewall = true;
  capSysAdmin = true;
  settings = {
    sunshine_name = "erebos";
    port = 47989;
    max_bitrate = 0;
    stream_audio = true;
    upnp = false;
    ping_timeout = 10000;
    min_threads = 2;
  };
};
```

- Notes: the NixOS module renders `settings` into the Sunshine configuration file (web UI is
  disabled when `settings` is set) and manages firewall ports via `openFirewall`; `capSysAdmin`
  is required for DRM/KMS screen capture.

- Acceptance: Moonlight discovers the server on LAN, pairs via Web UI PIN, streams a game
  with Intel Arc A380 (VAAPI) encoding.

## External references

- [Sunshine](https://github.com/LizardByte/Sunshine)
- [NixOS `services.sunshine` module](https://github.com/NixOS/nixpkgs/blob/901c54c08bdc20e00c2b3c73716b97497b1b3c74/nixos/modules/services/networking/sunshine.nix)
- [Sunshine configuration](https://docs.lizardbyte.dev/projects/sunshine/latest/md_docs_2configuration.html)

## Related ideas and dependencies

- [[docs/erebos/index]] · [[docs/erebos/containers]] · [[docs/erebos/services]] · [[ideas/github-pr-governance]]
