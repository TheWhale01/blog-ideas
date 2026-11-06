---
title: Migrate to systemd stage-1 initrd
created: 2026-11-04
updated: 2026-11-04
type: todo
tags: [erebos, todo, nixos, fix]
sources: ["erebos:sys/configuration.nix"]
---

# Migrate to systemd stage-1 initrd (remove scripted-initrd opt-out)

- **Status**: Not started
- **Priority**: High — hard deadline: **NixOS 26.11 becomes stable 2026-11-30** and *deletes* the
  scripted initrd; the flake pins `nixpkgs-unstable`, so the removal is already landing in the pin.
  The build is already warning; after removal it becomes a build failure or worse.
- **Parent**: [[todo/index]]

## Root cause of the warning

`sys/configuration.nix` line ~52 explicitly opts out of the new default:

```nix
boot.initrd.systemd.enable = false;   # ← this is what triggers the eval warning
```

NixOS 26.05 "Yarara" made **systemd Stage 1 the default initrd**; the legacy scripted one is
deprecated and removed in 26.11. There is **no comment or commit message found explaining the opt-out**
— first task is `git log -S "boot.initrd.systemd" sys/configuration.nix` to check whether it papered
over a real boot problem.

## Why Erebos should be a boring migration

Audited the boot surface in `sys/disko/prod.nix` + `stage.nix`:

- Root disk: plain GPT → vfat ESP (`/boot`) + **btrfs root, no LUKS, no LVM**, virtio `/dev/vda`
  (prod, i.e. a VM — console recoverable via `virsh console` on the qemu host) / `/dev/sda` (stage).
- `/data` mounts `/dev/mapper/whale--nas-data` — an external LVM volume **defined nowhere in the
  flake** (pre-opened outside NixOS boot, e.g. host pass-through); the scripted initrd never
  opened it, so systemd Stage 1 changes nothing for it. Just verify it still mounts after reboot
  (a stale `.mount` unit dependency could mask a change).
- No initrd-time agenix secrets, no network root, no SSH boot unlock.

If the opt-out was cargo-culted or covers only the `/data` mapper, the fix is one deleted line.

## Steps

- [ ] Find the opt-out's origin (`git log -S`) and note it here before touching anything
- [ ] Delete `boot.initrd.systemd.enable = false;` (default true now applies)
- [ ] `nix run .#dry-build` (or `nixos-rebuild dry-build`) — closure must evaluate warning-free
- [ ] **Stage first** if possible (`erebos-stage` env exists): reboot, confirm cold boot works
- [ ] Prod: `nix run .#apply` (or `boot` + reboot), then verify:
  - [ ] reaches login, `systemctl is-system-running`
  - [ ] `findmnt /data` → ext4 from `whale--nas-data` still mounted
  - [ ] `journalctl -b -p err` clean; `bootctl status` fine (systemd-boot unchanged)
- [ ] Rollback plan (free): pick previous generation in the systemd-boot menu, then re-add the line
  — the escape hatch exists **until 2026-11-30**, after that only "fix forward" works
- [ ] Bonus: with systemd Stage 1, the root mount gets proper device-unit ordering and the
  `rescue`/`emergency` targets; consider also dropping `x-systemd.device-timeout` worries later

## Related

- [[docs/erebos/architecture]] — disko layout
- [[docs/erebos/tutorials]] — §4 secret rotation unaffected; consider adding a "boot migration" note
- [[todo/backups]] — boot-path changes are exactly why restore tests matter

## References (verified 2026-11-04)

- NixOS 26.05 release notes (systemd stage-1 default, deprecation, LUKS `.device` caveat):
  https://nixos.org/manual/nixos/stable/release-notes.html#sec-release-26.05
- Deprecation context (impermanence issue tracking scripted-initrd removal):
  https://github.com/nix-community/impermanence/issues/325
- Systemd stage-1 initrd option docs:
  https://search.nixos.org/options?query=boot.initrd.systemd.enable
