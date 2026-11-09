---
title: CleanYourTV
created: 2026-11-07
updated: 2026-11-07
type: idea
tags: [privacy, android-tv, adb, homelab, jellyfin]
sources: ["https://github.com/Gathin23/cleanyourtv (owner share 2026-11-07)"]
---

# CleanYourTV

Debloat **Android TV / Google TV sets over Wi-Fi** and replace the spyware-laden stock launcher
with a clean, Apple-TV-style home screen. [Repo](https://github.com/Gathin23/cleanyourtv) ·
[Site](https://cleanyour.tv). Created 2026-10, ~55k stars (went viral fast).

## What it does (three steps, no root)

1. **Backup** the installed-package list + settings.
2. **Disable** manufacturer telemetry, ad services, promoted content rows and preloaded junk via
   `pm disable-user` over **wireless ADB** — *nothing is uninstalled*; a restore script undoes every
   step. Bloat differs per brand (Xiaomi, Sony Bravia, TCL, Fire TV…), so the tool discovers yours
   and asks before touching anything you might actually use.
3. **Install** a small native launcher and make it the default home.

Workflow is prompt-first: the intended path is handing its prompt to a coding agent (~15 min),
with a manual path too. TV brands that aren't Android (Samsung/LG/Roku/VIDAA) get a companion
"turn tracking off with the remote" guide instead — the tool itself can't run there.

## Homelab fit

- Post-debloat, a clean Android TV is the ideal thin client for **[[docs/erebos/services]] Jellyfin**
  (and anything else served from Erebos) — stock launches otherwise shoveload sponsored rows at you
  sourced from the TV's telemetry.
- Pairs philosophically with the fleet's privacy posture: same instinct as the Netbird/WireGuard
  choices and [[ideas/Deezify]] — own your media path, minimize vendor cloud.

## Caveats & security notes (worth knowing before running it)

- **Typosquat warning from the project itself**: official sources are `cleanyour.tv` + the repo only;
  `cleanyourtv.com`, `clean-your-tv.net` and any "app store" CleanYourTV apps are NOT theirs (#1004).
  Trust nothing distributed outside the repo — ironic but important for an adb-driven tool.
- Requires enabling **Developer options → network ADB** on the TV, i.e. an open debug port on your
  LAN for the duration. Trusted LAN ([[docs/erebos/architecture]]) + do it once + disable after.
- **Firmware updates can re-enable bloat** — the repo covers re-run/restore behavior; expect
  occasional re-checks.
- Very young (Oct 2026) + heavily vibe-coded + 260+ open issues: it's popular, not mature — the
  reversible `disable-user` design is the mitigating safety property. License is "Other" — check
  before any redistribution assumption.

## Related

- [[docs/erebos/services]] — Jellyfin (the client it improves)
- [[ideas/Deezify]] — same "own the media pipeline" instinct
- [[docs/erebos/architecture]] — trusted LAN context (podman0/wt0)
