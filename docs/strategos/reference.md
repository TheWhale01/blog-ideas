---
title: Strategos Reference
created: 2026-11-07
updated: 2026-11-07
type: reference
tags: [strategos, nixos, homelab, security]
sources: ["https://github.com/TheWhale01/strategos (sys/ @ main, read 2026-11-07)"]
---

# Strategos — Reference

## Options quick table

| Option | Value |
|---|---|
| `networking.hostName` | `strategos` |
| `networking.enableIPv6` | `false` (+ `boot.kernelParams = [ "ipv6.disable=1" ]`) |
| `networking.networkmanager.enable` | true (`athena` in `networkmanager` group) |
| `networking.firewall.enable` | true (no explicit `allowedTCPPorts` in repo → base 22 only via services.openssh's `openFirewall`? verify: `services.openssh` default does not open the firewall — the shared `nixos-modules` system module presumably allows it) |
| `boot.loader.generic-extlinux-compatible.enable` | true (grub off) |
| `boot.kernelModules` | `r8712u` (USB Wi-Fi dongle) |
| `swapDevices` | `/var/lib/swap/swapfile`, 4096 MiB (created out-of-band) |
| `services.openssh.settings` | no passwords, no kbd-interactive, no root login |
| `services.create_ap.settings` | `enu1u1u1` → `wlan0`, SSID `Chambre Hugo`, plaintext WPA2 passphrase |
| `nix.settings.trusted-users` | `root`, `athena` |
| `time.timeZone` | `Europe/Paris` |
| `system.stateVersion` | `25.05` (home-manager: `25.05`) |
| nixpkgs pin | `release-25.11` **stable** (Erebos runs nixpkgs-unstable) |

## Access matrix

| From | To | Auth |
|---|---|---|
| laptop / Erebos | `ssh athena@strategos` | ed25519 keys in `authorizedKeys` (password auth off) |
| Wi-Fi clients | SSID `Chambre Hugo` | WPA2, weak committed passphrase ⚠️ |
| strategos → LAN | wired `enu1u1u1` via NM/DHCP | — |

## Security notes (2026-11-07)

- ⚠️ **AP passphrase is a short weak pattern in plaintext in a public repo.** Fix when convenient:
  strong random PSK + agenix on the Pi (which it lacks today — piggyback on the keyring work in
  [[todo/age-keys-to-strategos]], which will introduce agenix there anyway).
- Version skew: 25.11-stable pin vs Erebos' unstable — fine for a receiver box, but remember borg
  client/server version compatibility when [[todo/backups]] lands (Pi's stable borg may lag).
- No disk encryption (SD image); acceptable given nothing sensitive lives on it *yet* — stays true
  only if the keyring backup stores age-**encrypted** material (it does by design).
- 1 GiB RAM + 4 GiB swapfile: fine for sshd+hostapd+borg receiver; do not plan heavy work here.
- Firewall is on with no documented inbound rule besides SSH — verify with `sudo nft list ruleset`
  on the box before exposing a borg-rsyncd or anything else.

## Planned roles

1. **Keyring host** — offline copy of agenix identity keys → [[todo/age-keys-to-strategos]]
2. **Backup receiver** — borg repo over SSH for vaultwarden/immich/nextcloud data → [[todo/backups]]

## Related

- [[docs/strategos/index]] / [[docs/strategos/services]] — overview & per-module docs
- [[docs/erebos/index]] — the server it supports
