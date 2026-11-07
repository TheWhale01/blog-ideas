---
title: Reconcile disko prod device paths
created: 2026-11-07
updated: 2026-11-07
type: todo
tags: [erebos, todo, nixos, fix, infrastructure]
sources: ["live /sys/block inventory 2026-11-07", "erebos:sys/disko/prod.nix"]
---

# Reconcile disko prod device paths

- **Status**: Not started
- **Priority**: High — latent reinstall footgun, low daily-ops risk
- **Parent**: [[todo/index]]

## The mismatch (found 2026-11-07)

`sys/disko/prod.nix` declares the root disk as **`/dev/vda`** (virtio), but the production machine is
**bare metal** (owner-confirmed; live evidence: `systemd-detect-virt` = none, AMD Ryzen 5 5600, six
ATA disks `sda`–`sdf` in `/sys/block`, **no virtio block devices at all**). The config was almost
certainly written when prod ran as a VM (consistent with the `whale-nas-data` legacy naming story)
and was carried over to the physical box without updating the paths.

Why it matters: the running system boots fine (device paths are only consumed at install time / by
fileSystems resolution that evidently still works), **but** re-running `disko --mode disko` with this
file would target a nonexistent device — and worse, if device enumeration ever shifted, stale paths
make the install procedure unsafe. This also sits under the planned disk work ([[ideas/Drive-Health-Monthly-Newsletter]]):
drive-replacement decisions need a trustworthy disk-role map first.

## Tasks

- [ ] Capture ground truth on the box (as root): `lsblk -o NAME,MODEL,SIZE,FSTYPE,MOUNTPOINTS` ·
      `pvs; vgs; lvs` (which disk(s) hold VG `whale`) · `findmnt -R /` · `cat /sys/firmware/efi/... ` or `bootctl status`
- [ ] Map: which sdX = btrfs root, which = whale PV for `/data`, roles of the remaining four disks
      (any spare/old NAS transplants?)
- [ ] Update `sys/disko/prod.nix` to the real devices (or switch to stable paths:
      `/dev/disk/by-id/ata-…` — survives enumeration changes)
- [ ] Record the resulting disk-role table into [[docs/erebos/architecture]] §5 (and mirror into the
      newsletter idea's inventory table)
- [ ] Sanity-check: `nix flake check` + rebuild; do **not** rerun disko unless intending reinstall

## Related

- [[docs/erebos/architecture]] — §5 disk layout (updated with the stale-path note)
- [[ideas/Drive-Health-Monthly-Newsletter]] — needs the per-disk role map
- [[docs/erebos/tutorials]] — §5 disko volumes tutorial
