---
title: Make the issues room read-only (bot-only writes)
created: 2026-11-07
updated: 2026-11-07
type: todo
tags: [erebos, matrix, element, permissions, todo]
sources: ["owner request 2026-11-07", "room v12 creator-PL semantics — see todo/monitoring-alerting v12 research"]
---

# Make the issues room read-only (bot-only writes)

- **Status**: Not started
- **Priority**: Low
- **Parent**: [[todo/index]]

Reminder (Element, manual — power levels are room state, not server config):

1. Open the **issues room** (in the Erebos space) → Settings → **Roles & permissions**
   (a.k.a. "Advanced → permissions").
2. Set the **Send messages** requirement (`events_default` for `m.room.message`) to **50**.
   Members below PL 50 can read but not write; alert traffic from the bot is unaffected
   once it has PL ≥ 50.
3. Grant **@erebot:matrix.thewhale.fr power level 50** (custom levels / moderators list).
   If the room is v12 and @erebot created it, it already holds infinite PL — skip.
4. Verify: a normal member account sees the composer disabled.

## v12 caveat (decide before acting)

Room version 12 makes the **creator's power level infinite and unchangeable** — the create
event's sender can ALWAYS post, regardless of any power-levels event. So:
- If **@whale created the room**: everyone else can be silenced, but *you* cannot be locked
  out of your own v12 room. Options: (a) accept creator-as-exception, or (b) create a fresh
  issues room **with @erebot joined first / created via the bot** so the bot is the creator
  (then you are PL-manageable at 0), swap it in the space, archive the old one.
- Check version & creator quickly: room → Settings → Advanced → room details (version);
  creator = sender of the initial `m.room.create` (first event / `/debug` in Element Web).

If the issues room is the alerts room from [[todo/monitoring-alerting]] (`!C-EIAS89…`,
server-less ID ⇒ v12), its creator identity decides case (a) vs (b) — verify before clicking.
