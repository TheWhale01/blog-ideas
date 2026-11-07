---
title: Strategos Services
created: 2026-11-07
updated: 2026-11-07
type: reference
tags: [strategos, nixos, homelab]
sources: ["https://github.com/TheWhale01/strategos (sys/ @ main, read 2026-11-07)"]
---

# Strategos — Services & modules

One page mirroring `sys/` of [TheWhale01/strategos](https://github.com/TheWhale01/strategos), file by
file. Everything below is read from the repo as of 2026-11-07.

## `sys/configuration.nix`

- Imports `hardware-configuration.nix`, `./services`, `packages.nix`.
- Boot: `boot.loader.grub.enable = false`, `generic-extlinux-compatible.enable = true` (Pi U-Boot
  extlinux), `boot.kernelModules = [ "r8712u" ]` (Realtek RTL8712-class USB Wi-Fi dongle driver),
  `boot.kernelParams = [ "ipv6.disable=1" ]`.
- Networking: `hostName = "strategos"`, `enableIPv6 = false`, NetworkManager enabled, firewall enabled.
- `swapDevices`: file `/var/lib/swap/swapfile`, 4096 MiB — must be created out-of-band (NixOS does not
  format it; a first-install step makes the file).
- `hardware.enableRedistributableFirmware = true`.
- Nix: experimental-features flakes; `trusted-users = [ root athena ]`.
- `users.users.athena`: normal, `wheel` + `networkmanager`, zsh, two `openssh.authorizedKeys` entries
  (ed25519 — laptop key and Erebos `hades@erebos` key).
- `programs.zsh.enable = true`; `time.timeZone = "Europe/Paris"`; `system.stateVersion = "25.05"`.

## `sys/services/ssh.nix`

`services.openssh.enable = true` with hardened settings: `PasswordAuthentication = false`,
`KbdInteractiveAuthentication = false`, `PermitRootLogin = "no"`. Key-only access via `athena`'s
authorized_keys.

## `sys/services/create_ap.nix`

Software access point bridging the wired uplink to Wi-Fi:

- `INTERNET_IFACE = "enu1u1u1"` (USB-Ethernet), `WIFI_IFACE = "wlan0"` (dongle),
- `SSID = "Chambre Hugo"`, WPA2 `PASSPHRASE` set **in plaintext in the config** ⚠️
- systemd unit ordering patched: `create_ap` runs `after`/`wants`
  `sys-subsystem-net-devices-enu1u1u1.device` (waits for the uplink to appear).

> ⚠️ **Secrets finding (2026-11-07):** the repo is public and the AP passphrase is a short weak
> pattern committed in clear text. Anyone can join the hotspot; blast radius is "guest LAN segment"
> rather than fleet infrastructure, but it should move to agenix + a strong passphrase. See
> [[docs/strategos/reference]] § security notes. (Passphrase deliberately not reproduced here.)

## `sys/packages.nix`

`vim, libraspberrypi, wget, git, btop, wakeonlan, fastfetch, usbutils, gnumake` — note `wakeonlan`:
the Pi is (at least partly) the box that *wakes* other machines on the LAN (1 GiB-RAM Pi kept useful
even when Erebos is down).

## `sys/home.nix` / `sys/home-manager/`

Home Manager for `athena` (stateVersion 25.05), which imports `inputs.modules.nixosModules.homeManager`
— i.e. the **shared `nixos-modules` repo** customization also used on Erebos' `hades` user; the
flake sets `useGlobalPkgs`, `useUserPackages`, and `backupFileExtension = "bkp"`.

## `sys/hardware-configuration.nix`

Generated file, untouched: root fs ext4 at `/dev/disk/by-uuid/4444…8888` (the NixOS SD-image
placeholder-UUID → resize/install step), `usbhid` in initrd, no modules otherwise (Pi config).

## Related

- [[docs/strategos/index]] — machine overview
- [[docs/strategos/reference]] — options & security notes
- [[docs/erebos/services]] — the server's service doc (same style)
