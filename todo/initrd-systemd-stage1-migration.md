---
title: Migrate to systemd stage-1 initrd
created: 2026-11-04
updated: 2026-11-04
type: todo
tags: [erebos, todo, nixos, fix]
sources: ["erebos:sys/configuration.nix"]
---

# Migrate to systemd stage-1 initrd (remove scripted-initrd opt-out)

- **Status**: In progress (2026-11-07: stage env migrated cleanly; prod blocked by assertion — root-caused, fix known)
- **Priority**: High — hard deadline: **NixOS 26.11 becomes stable 2026-11-30** and *deletes* the
  scripted initrd; the flake pins `nixpkgs-unstable`, so the removal is already landing in the pin.
  The build is already warning; after removal it becomes a build failure or worse.
- **Parent**: [[todo/index]]

## Root cause of the warning — SOLVED (archaeology done 2026-11-07)

The opt-out existed to support the LVM-for-`/data` hack, but the hack itself was inert:

- `sys/configuration.nix`: `boot.initrd.systemd.enable = false;` ← the warning trigger
- `sys/disko/prod.nix`: `boot.initrd.postDeviceCommands = lib.mkAfter ''"lvm vgchange -ay"'';` ← added
  to support LVM (`/dev/mapper/whale--nas-data`), and exactly what systemd stage 1 rejects with a
  hard assertion (verified on prod 2026-11-07; stage passed — it has no LVM: plain sda GPT+btrfs)
- **The hack never did anything**: (1) quoting bug — `''"…"''` makes the shell execute one quoted
  token → command-not-found, swallowed by scripted initrd; (2) redundant anyway — the scripted
  `stage-1-init.sh` already runs `lvm vgchange -ay` itself inside its device-wait loop
- **systemd stage 1 covers it declaratively**: in the pinned nixpkgs (`tasks/lvm.nix`),
  `boot.initrd.services.lvm.enable` defaults to `boot.initrd.systemd.enable && services.lvm.enable`,
  with `services.lvm.enable = true` by default → LVM tools + udev rules + activation units enter the
  initrd automatically once stage-1 is on. No replacement config needed.

## Fix = two deletions

```nix
# sys/configuration.nix — delete:
boot.initrd.systemd.enable = false;
# sys/disko/prod.nix — delete the whole block:
boot.initrd.postDeviceCommands = lib.mkAfter ''
  "lvm vgchange -ay"
'';
```

## Steps

- [x] Find the opt-out's origin — done: prod-only LVM hack for disko `data` lvm-type device, inert
- [x] Stage env migrated & rebooted — no issues (no LVM there, assertion never fires)
- [ ] Delete the two blocks above
- [ ] `nix run .#dry-build` — assertion gone, eval warning-free
- [ ] Prod: `nix run .#apply`, then **cold reboot** (stage-1 only takes effect at boot); previous
      generation stays in the systemd-boot menu as rollback
- [ ] Verify: `findmnt /data` → `/dev/mapper/whale--nas-data` ext4 · `systemctl status lvm2-monitor` ·
      `journalctl -b -p err` clean · Traefik/Authentik/Nextcloud come up
- [ ] Note: prod root is plain btrfs on virtio `/dev/vda` (no LUKS/LVM in the boot path) — residual
      risk minimal; `/data` is the only LVM consumer

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
