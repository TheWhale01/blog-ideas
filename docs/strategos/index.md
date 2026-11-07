---
title: Strategos Documentation
created: 2026-11-07
updated: 2026-11-07
type: reference
tags: [strategos, nixos, homelab]
sources: ["https://github.com/TheWhale01/strategos (sys/ @ main, read 2026-11-07)"]
---

# Strategos

**Strategos** is the fleet's Raspberry Pi **3B+** (`aarch64-linux`) — a small, always-on NixOS box on
Erebos' LAN, configured declaratively by [TheWhale01/strategos](https://github.com/TheWhale01/strategos).

Its current job is to bridge the home network into an isolated access point ("guest hotspot"
pattern): it uplinks over USB Ethernet and serves a software Wi-Fi AP. Its future job (agreed
2026-11-07): become the **off-box keyring** for Erebos — see [[todo/age-keys-to-strategos]] and
[[todo/backups]].

## Machine facts

| | |
|---|---|
| Hardware | Raspberry Pi 3B+; USB Wi-Fi dongle (kernel module `r8712u`) + USB-Ethernet adapter (`enu1u1u1` internet uplink, `wlan0` AP) |
| OS | NixOS, flake pinned to `nixpkgs` **release-25.11** (stable — older channel than Erebos' unstable pin) |
| Boot | `generic-extlinux-compatible` bootloader (no grub); root = ext4 on SD-card-image UUID layout |
| Storage | 4 GiB **swapfile** at `/var/lib/swap/swapfile` (3B+ has 1 GiB RAM) |
| Network | NetworkManager, **IPv6 disabled** (kernel param + option), NixOS firewall enabled |
| Users | `athena` (wheel + networkmanager, zsh); stateVersion `25.05` |
| Access | SSH, key-only (two authorized ed25519 keys: from the owner's laptop and from Erebos) |
| Timezone | Europe/Paris |

## Flake inputs

| Input | URL | Notes |
|---|---|---|
| nixpkgs | `github:nixos/nixpkgs/release-25.11` | stable channel; `allowUnfree`/`allowBroken` set in pkgsFor |
| home-manager | `github:nix-community/home-manager/release-25.11` | follows nixpkgs; manages `athena` |
| modules | `github:TheWhale01/nixos-modules` | **shared with Erebos** — same base system module (`nixosModules.system`; `homeManager` module imported by home.nix) |

## Layout

```
flake.nix                  # single nixosConfiguration: strategos (aarch64-linux)
sys/
├── configuration.nix      # imports services/ + packages.nix + hardware-config
├── hardware-configuration.nix
├── packages.nix           # vim, libraspberrypi, wget, git, btop, wakeonlan, fastfetch, usbutils, gnumake
├── home.nix + home-manager/   # athena via shared nixos-modules homeManager module
└── services/
    ├── ssh.nix            # hardened OpenSSH
    └── create_ap.nix      # software AP (see docs/strategos/services)
```

## Pages

- [[docs/strategos/services]] — per-module documentation of every `sys/` file
- [[docs/strategos/reference]] — options table, access matrix, security notes, planned roles

## Related

- [[docs/erebos/index]] — the server strategos supports
- [[todo/age-keys-to-strategos]] — planned keyring role
- [[todo/backups]] — planned backup-receiver role (borg over SSH)
