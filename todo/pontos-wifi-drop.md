---
title: Pontos Wi-Fi dropouts
created: 2026-11-16
updated: 2026-11-16
type: todo
tags: [erebos, nixos, todo, improvement, networking]
sources: ["owner request 2026-11-16"]
---

# Pontos Wi-Fi dropouts

- **Status**: Not started
- **Priority**: Medium
- **Parent**: [[todo/index]]

## Problem

The Wi-Fi card on **pontos** — the daily-driver laptop (Framework 13, AMD Ryzen AI 9 HX 370, Fedora 45) —
sometimes drops the connection to the Wi-Fi network it is currently connected to.

## Context and notes

- Pontos is the owner's personal daily-driver laptop, not a server host.
- Symptom: intermittent loss of the active Wi-Fi link (adapter / driver / supplicant level).
- Wi-Fi adapter (from `lspci` on pontos): MEDIATEK Corp. MT7925 (RZ717) Wi-Fi 7 160MHz, kernel driver in use: mt7925e.

## Research — known issues (MT7925 / RZ717, from web 2026-11-16)

- Kernel 7.1.x mt76 driver regression: MT7925 Wi-Fi 7 MLO / 5 GHz association breakage (CachyOS
  report; upstream fix `b2f5def2f133` merged for 7.2, not backported to 7.1.y). Workaround: linux-lts
  kernel or full `mt7925e` module reload.
- Fedora: newer kernels (Fedora 43 / updated 42) hard-block WLAN via rfkill — fix `sudo rfkill unblock wlan`.
- Firmware regression: mt7925 blob `20260813113118` causes severe low-RSSI 5 GHz loss; pin to `20260605184805`
  (linux-firmware commit `bd1c66cf…`) and reload.
- Reports of drops / latency spikes / signal fluctuation (Fedora 44 + MT7925/RZ717): latency spikes
  4ms→160ms, signal loss; possibly driver/firmware-related.

## Task checklist

- [ ] Capture diagnostics: `journalctl -u NetworkManager` (link-loss / supplicant events),
      `journalctl -k` (kernel + firmware messages).
- [ ] Identify the wireless adapter (`lspci | grep -i -E 'network|wireless'` / `rlinktool`).
- [ ] Check mitigations: linux-firmware / kernel update, Wi-Fi power management, module reload.
- [ ] Decide on fix: workaround (config) vs hardware / module change.

## Related

- [[ideas/chezmoi-pontos]] — pontos machine context (Fedora migration, chezmoi dotfiles).
- [[todo/index]] — todo hub.

## History

- 2026-11-16: added per owner request (Wi-Fi card sometimes drops the Wi-Fi connection).
- 2026-11-16: added Wi-Fi card model from `lspci` output (MT7925 / RZ717, Wi-Fi 7 160MHz, driver mt7925e).
- 2026-11-16: added web research findings for MT7925 / RZ717 (kernel 7.1.x mt76 regression, rfkill, firmware pin).
