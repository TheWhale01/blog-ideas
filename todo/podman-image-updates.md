---
title: Automate Podman Image Updates
created: 2026-10-15
updated: 2026-10-22
type: todo
tags: [nixos, erebos, todo, improvement, automation]
sources: []
---

# Automate Podman Image Updates

- **Status**: Partial (basic `update-containers` service exists)
- **Priority**: High
- **Parent**: [[todo/index]]

## Description
The current daily oneshot in `sys/containers/default.nix` pulls *every* image and restarts all `podman-*` units regardless of whether anything changed. No validation, no notifications.

## Tasks
- [ ] Replace with Podman-native `podman auto-update` (or hardened script)
- [ ] Pull/restart only if the image digest changed
- [ ] Validate image signatures where registries publish them
- [ ] Keep the old container running on failure (no restart loops)
- [ ] Add Matrix notifications on failures + weekly "newest digests" report

## Detailed notes
Label containers with `io.containers.autoupdate=image` (or `registry` for digest pinning) and run `podman auto-update --dry-run` first to verify only changed images trigger restarts. For systemd-managed podman units use `--type=systemd`.

Signatures: cosig/sigstore support is patchy across the GHCR images in use — treat as best-effort (validate for images that publish them, e.g. linuxserver.io).

Failure path: on error, keep the old container running, emit a Matrix notification (webhook secret already exists). Make updates observable, not silent, via a weekly digest report to the same channel.

No dependencies — can land before [[todo/monitoring-alerting]] (which should then suppress transient-restart alerts).

## Related
- [[docs/erebos/containers]]
- [[todo/monitoring-alerting]]
