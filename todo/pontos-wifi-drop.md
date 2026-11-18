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
