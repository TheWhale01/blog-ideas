---
title: Backup agenix identity keys to strategos
created: 2026-11-07
updated: 2026-11-07
type: todo
tags: [erebos, todo, security, infrastructure]
sources: ["owner decision 2026-11-07"]
---

# Backup agenix identity keys to strategos (Raspberry Pi)

- **Status**: Not started
- **Priority**: High
- **Parent**: [[todo/index]]

## Why this exists

There is **no offline copy of the agenix identity keys today** (owner-confirmed 2026-11-07). Losing
them makes every `.age` file permanently unreadable — SSO client_secrets, SMTP tokens, VPN configs,
DB creds — and would also lock out any future backup's own encryption. This is the single
highest-severity latent risk on Erebos: silent until the day it isn't.

## Target

**strategos** — the owner's Raspberry Pi **3B+**, aarch64 NixOS, same LAN ([config repo](https://github.com/TheWhale01/strategos): flake pinning `nixpkgs release-25.11` stable, user `athena`,
shares the `nixos-modules` input with Erebos). Current state relevant to this task: **SSH server up
with `athena` authorized keys** (transfer channel exists today), firewall enabled, NetworkManager +
DHCP (pin a static lease or use `strategos.local` mDNS so Erebos' job targets a stable address),
**no agenix on it yet** — option 2 below would require adding it.

## Design options (pick before implementing)

1. **Passphrase archive (simplest, recommended first pass):**
   `age -p` a tarball of the identity file(s) (default: `/var/lib/agenix-keys/id_ed25519.age`-equivalent
   — locate exactly via the flake's agenix `identities` config) → copy to the Pi at a fixed path,
   say `/srv/keyring/`. Store the passphrase somewhere *not* Erebos (password manager). Add a second
   copy on USB kept off the Pi for true offline.
2. **Pi as agenix recipient:** generate an age keypair on the Pi, commit its public key to the secrets
   repo as a recipient of a dedicated `agenix-master-keys.age` containing the identity material, and
   let the Pi periodically `sudo nixos-rebuild switch`-style decrypt-and-store. Elegant but couples
   the Pi's rebuild state to key availability — overkill for this scale.

## Tasks

- [ ] Locate the exact identity path(s) agenix uses on Erebos (`age.identities` in the flake / `/etc/agenix*`)
- [ ] Decide option 1 vs 2 (start with 1)
- [ ] Implement copy; verify **round-trip**: from a clean machine, decrypt the archive with only the
      passphrase and use the identity to decrypt one `.age` file — this is the drill that makes the
      backup real
- [ ] Document on the Pi (README in `/srv/keyring/` describing restore procedure)
- [ ] Add a reminder cadence (quarterly round-trip check) — candidate to fold into
      [[todo/monitoring-alerting]] or the weekly agent check-in

## Related

- [[docs/erebos/secrets]] — rotation posture; this closes its "key-loss cliff" caveat
- [[docs/strategos/index]] — the Pi's own documentation (current config, security notes)
- [[todo/backups]] — same Pi doubles as the backup target for the 3 sensitive services
- **strategos config** — https://github.com/TheWhale01/strategos
