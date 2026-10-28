---
title: Quickshell Notification Client for quickconf
created: 2026-10-26
updated: 2026-10-26
type: idea
tags: [IT, development, homelab, creativity]
sources: [https://github.com/TheWhale01/quickconf]
---

# Quickshell Notification Client for quickconf

**Status**: unimplemented

## Context

[quickconf](https://github.com/TheWhale01/quickconf) is the modular [Quickshell](https://quickshell.outfoxxed.me) + QML status bar used on **pontos** (Hyprland on Fedora) — Catppuccin Mocha styled, `Global.qml` singleton theming, native SNI tray, Hyprland workspace animations. It has everything a desktop shell needs **except a notification daemon**: pontos currently has no (or a generic, off-theme) notification server. The idea: implement a custom notification client *inside the quickconf codebase*, styled identically to the bar.

## Good news: quickshell already implements the hard part

Verified against upstream source (2026-10-26): quickshell ships a dedicated module
**`Quickshell.Services.Notifications`** — *"Types for implementing a notification daemon"*.
It owns `org.freedesktop.Notifications` on the session bus and parses the entire protocol
(`src/services/notifications/server.cpp`, the freedesktop interface XML, D-Bus image decoding);
you only write the **QML UI**. Confirmed API surface (from `notification.hpp`/`server.hpp` on master):

- `NotificationServer` — the daemon entry point; exposes `trackedNotifications()` as an
  `ObjectModel<Notification>` (bind a `Repeater` straight to it) and a `notification(Notification)`
  signal for "just arrived" pop triggers, plus `NotificationClosed(id, reason)` / `NotificationReplied(id, text)`.
- `Notification` — `id`, `appName`, `appIcon`, `summary`, `body`, `urgency` (enum), `image`,
  `actions` (list of `NotificationAction`), `hasInlineReply`, `inlineReplyPlaceholder`,
  `expireTimeout`, `resident`, `transient`, `hints`, `desktopEntry`; methods `expire()`, `dismiss()`,
  `sendInlineReply(text)`.
- Inline replies, actions and image payloads are therefore **first-class**, not a stretch goal.

Check the exact registration/version in the quickshell reference docs on
[quickshell.outfoxxed.me](https://quickshell.outfoxxed.me) and inside your pinned nix flake input
(the quickconf dev-shell already runs the QML LSP against the right version, so property
completion will guide you).

## Design sketch

- **Ownership**: registering the QML `NotificationServer` takes the bus name — if `mako` (or another daemon) currently owns it on pontos, disable it first (`systemctl --user mask mako` / drop autostart; check with `busctl --user status org.freedesktop.Notifications`).
- **Shape**: `src/components/Notifications.qml` — an anchor-type popup window (bottom-right, above the bar) whose `Repeater` binds to `NotificationServer.trackedNotifications()`; new-arrival animation on the `notification` signal; a per-card `MouseArea`/`Button` calling `dismiss()`/`expire()`, action buttons from `notification.actions`, reply field when `hasInlineReply`.
- **Look**: bottom-right column, Catppuccin Mocha from `Global.qml` (`#11111b` surface, `#f5c2e7` accent), entry animation echoing the Bezier feel of `Workspaces.qml`, urgency → accent stripe; collapse beyond ~5.
- **DND**: not part of the upstream module — implement as a `Global.qml` property gating visibility (senders still succeed; messages keep being tracked), surfaced as a bell widget beside `Caffeine.qml` (same idiom).
- **Hyprland nicety**: layer rule so popups never steal focus.

## Tasks

- [ ] Check what currently owns `org.freedesktop.Notifications` on pontos; disable it cleanly
- [ ] Spike: minimal `Notifications.qml` — `NotificationServer` + a plain Window with a Repeater, `notify-send` round-trip
- [ ] MVP UI: stacking, dismissal, urgency styling, app icons, markup-safe Text (`textFormat: Text.RichText` or strip)
- [ ] Actions + inline reply wiring (already exposed by the module)
- [ ] DND state on `Global.qml` + bell toggle widget next to Caffeine
- [ ] Hyprland layer rule (no focus steal) + startup ordering (daemon before senders)
- [ ] Update quickconf README feature list (it advertises Waybar-replacement parity)
- [ ] Document final setup as an implemented service page (per wiki lifecycle)

## Gotchas

- Timeout edge cases: `expireTimeout <= 0` means never-expire; cap the visible stack (bind list length) so a chatty sender can't fill the screen.
- The daemon must start before senders cache the bus name; run quickconf as the Hyprland startup service, not a late `.desktop` autostart.
- Icons: Qt needs the icon theme resolvable in the quickshell env — the flake may need `adwaita-icon-theme` added alongside `hicolor`.
- Be liberal in what you accept: some senders send odd markup/empty icons; a malformed notification must never take down the bar process (guard bindings; consider a separate quickshell instance if the crash coupling ever bites — noted as alternative shape).

## Related

- [[ideas/Chezmoi-Pontos]] — same machine; notification daemon config would also live in the dotfiles repo
- [[index]] — this vault

## External references

*(all URLs curl-verified 2026-10-26 — note: quickshell's old `quickshell.dev` vanity domain does not resolve; the canonical site is `quickshell.outfoxxed.me`)*

- **quickshell** — site: https://quickshell.outfoxxed.me · GitHub mirror: https://github.com/quickshell-mirror/quickshell · primary forge: https://git.outfoxxed.me/quickshell/quickshell
- **quickshell Notifications module (upstream source, read before coding)** — https://github.com/quickshell-mirror/quickshell/tree/master/src/services/notifications (`module.md` = "Quickshell.Services.Notifications — Types for implementing a notification daemon")
- **quickconf (the config to extend)** — https://github.com/TheWhale01/quickconf
- **Desktop Notifications spec** — https://specifications.freedesktop.org/notification-spec/latest/
- **mako (reference implementation / prior art)** — https://github.com/emersion/mako
- **Qt QML docs** — https://doc.qt.io/qt-6/qtqml-index.html
- **Hyprland wiki (layer rules, focus)** — https://wiki.hypr.land/
