---
title: Automatically re-create movie libraries like Disney+ ones in Jellyfin
created: 2026-11-12
updated: 2026-11-12
type: idea
tags: [idea, active, media, collections, automation, streaming]
sources: []
---

# Automatically re-create movie libraries like Disney+ ones in Jellyfin

## Status

Proposed — researched 2026-11-12; not yet implemented.

## Problem statement

- The owner runs a Jellyfin movie library (`jellyfin.thewhale.fr`, see
  [[docs/erebos/services]]) and wants Disney+-style organization of it.
- Disney+ style means themed groupings of movies — essentially per-franchise /
  per-studio "libraries" (Disney, Pixar, Marvel, Star Wars, …) rendered as
  curated rows/collections, without hand-curating every movie.
- Recreating or refreshing those groupings today is either manual or ad-hoc
  plugin-driven; there is no standing automation to rebuild them from the
  existing library.

## Research findings (2026-11-12)

- [[docs/erebos/services]]: Jellyfin media server via `pkgs-unstable.jellyfin`;
  daily `get_xmltv` pulls XMLTV into `jellyfin.dataDir`.
- TMDB Box Sets plugin ([jellyfin-plugin-tmdbboxsets](https://github.com/jellyfin/jellyfin-plugin-tmdbboxsets)):
  automatically creates movie box sets (franchise groupings) from TMDb collections.
- Jellyfin-Auto-Collections
  ([ghomasHudson/Jellyfin-Auto-Collections](https://github.com/ghomasHudson/Jellyfin-Auto-Collections)):
  auto-creates collections from IMDb / Letterboxd / TMDb / Trakt / mdblist /
  Criterion Channel / Jellyfin API queries / Radarr–Sonarr tags; cron-scheduled.
- Jellyfin SmartLists plugin
  ([jyourstone/jellyfin-smartlists-plugin](https://github.com/jyourstone/jellyfin-smartlists-plugin)):
  rule-based playlists/collections (genre contains "Action", played = false, …)
  that auto-update on library updates or via scheduled tasks.
- Manual path: create collection → add movies; library option "Automatically add
  to collection" (which auto-creates a stray `Collections` library — see
  [jellyfin#16926](https://github.com/jellyfin/jellyfin/issues/16926) where the
  auto-created `Collections` library keeps reappearing).
- Synthesis: "Disney+-like" can be approximated today with TMDB Box Sets
  (franchise groupings) and/or Auto Collections / SmartLists (dynamic themed
  collections); "automatically re-create movie libraries" would most plausibly
  mean a Hermes-driven scheduled reconcile that rebuilds these collections from
  the existing library.

## Proposed solution

1. Enable TMDB Box Sets (or Auto Collections / SmartLists) on the existing movie library.
2. Define the Disney+-style grouping rules (studio/franchise → collection).
3. Add a Hermes cron/scheduled task to re-run the collection sync when the
   library changes (or on a schedule), so collections are re-created/refreshed
   automatically.

## Caveats & open questions

- TMDB Box Sets handles movie box sets; TV season grouping is a separate setting.
- The auto-created `Collections` library behavior is buggy
  ([jellyfin#16926](https://github.com/jellyfin/jellyfin/issues/16926)).
- "Re-create movie libraries" vs "re-create collections" is an open scope decision:
  per-studio libraries (new libraries) vs. themed collections within one library.

## External references

- [TMDB Box Sets plugin](https://github.com/jellyfin/jellyfin-plugin-tmdbboxsets)
- [Jellyfin Auto Collections](https://github.com/ghomasHudson/Jellyfin-Auto-Collections)
- [Jellyfin SmartLists plugin](https://github.com/jyourstone/jellyfin-smartlists-plugin)
- [Jellyfin collections auto-creation bug](https://github.com/jellyfin/jellyfin/issues/16926)
- [JellyWatch: Jellyfin Collections and Smart Playlists](https://jellywatch.app/blog/jellyfin-collections-smart-playlists-organize-library-2026)

## Related ideas and dependencies

- [[docs/erebos/services]] · [[docs/erebos/containers]] · [[ideas/github-pr-governance]]
