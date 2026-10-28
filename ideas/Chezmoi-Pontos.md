---
title: Chezmoi for pontos dotfiles
created: 2026-10-26
updated: 2026-10-26
type: idea
tags: [IT, development, automation, homelab]
sources: []
---

# Chezmoi for pontos dotfiles

**Status**: unimplemented

## Context

**Pontos** is the laptop. It ran NixOS for a long time, but NixOS proved painful for developing small projects (dev-shell friction, overlay archaeology for random npm/pip tooling, etc.), so it moved to **Fedora**. The price of that trade: losing the declarative config that made NixOS wipes survivable — `configuration.nix` + `home-manager` meant "rebuild and you're back." The missing piece on Fedora is **home-manager**; the goal is to recreate that "wipe and recover" property for dotfiles with **chezmoi**.

## Why chezmoi

- Single Go binary, no runtime daemon — unlike a Nix stack it won't rot on a fast-moving Fedora.
- Git-native: dotfiles live in a normal repo (`TheWhale01/dotfiles`), versioned and reviewable.
- **Templating**: one repo can serve multiple machines via `chezmoi.toml` config + `.tmpl` files (pontos now, future laptop for free).
- **Age encryption**: encrypted files (`chezmoi age encrypt`) fit the household's existing agenix/age habits (see [[docs/erebos/secrets]]) — same key format, familiar workflow.
- Safe by design: `chezmoi diff`/`status`, managed files only, ignores by default vs allow-list `.chezmoiignore`.

Official: [chezmoi docs](https://www.chezmoi.io) · [GitHub repo](https://github.com/twpayne/chezmoi) · [age](https://github.com/FiloSottile/age) / [age.dev](https://age.dev) · [chezmoi age encryption guide](https://www.chezmoi.io/user-guide/encryption/age/)

## Alternative worth noting: home-manager *is* portable

home-manager runs on any Linux with Nix installed — including Fedora. Keeping Nix just for hm/home configs (not system-level, like NixOS) recovers most of the old workflow, and the existing Erebos/nixos-modules configs could partly be reused. **Trade-off vs chezmoi:**
- home-manager: stronger (manages packages & services config, one language), but reintroduces the Nix toolchain that made pontos painful — and its `flake.lock` staleness is real work.
- chezmoi: dotfiles only, zero extra runtime, trivially portable. If the goal is specifically "survive a wipe", chezmoi is the right weight class.
Decision: go with chezmoi; revisit hm only if config sprawl demands it.

See: [home-manager](https://github.com/nix-community/home-manager) / [official docs](https://nix-community.github.io/home-manager/).

## Implementation sketch

1. **Install** on pontos: `sh -c "$(curl -fsLS get.chezmoi.io)"` (or `dnf install chezmoi` — it's in Fedora repos).
2. **Init repo**: `chezmoi init --apply TheWhale01` pointing at a new private `dotfiles` GitHub repo (SSH — reuse the pattern from the blog-ideas repo).
3. **Structure** (kebab-case, machine-agnostic):
   ```
   dotfiles/
   ├── .chezmoiignore
   ├── .chezmoiscripts/
   │   └── run_onchange_install-packages.sh    # dnf install list, idempotent
   ├── dot_gitconfig.tmpl                      # {{ if eq .chezmoi.hostname "pontos" }} ...
   ├── dot_config/nvim/...
   ├── dot_zshrc (or .bashrc)
   ├── private_dot_ssh/config.tmpl
   └── encrypted_dot_config/age/…              # chezmoi-managed secrets via age
   ```
4. **Secrets**: use chezmoi's native `age` integration with a dedicated `chezmoi` age key (separate from agenix service keys), the key file itself bootstrapped from the offline key backup — do **not** reuse the SSH-derived identity for everything (see [[todo/secrets-management]] on Erebos for the same principle).
5. **Fedora specifics** that dotfiles alone won't cover — keep as `run_` scripts: dnf package set, flatpak list, `gnome-extension` enables, `gsettings` dumps (chezmoi handles files; gsettings/dconf need scripts or `dconf load`).
6. **Wipe-and-recover drill** (the whole point): fresh Fedora → install curl/git/age → one line: `sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply TheWhale01` → `chezmoi verify` clean.

## Tasks

- [ ] Create private `dotfiles` repo (GitHub, SSH)
- [ ] `chezmoi init` on pontos, import current dotfiles, commit
- [ ] Add machine template config for hostname `pontos`
- [ ] Set up chezmoi `age` key + encrypted secrets (gitconfig tokens, ssh config)
- [ ] Package/dconf/flatpak bootstrap scripts
- [ ] Do one dry-run restore test in a Fedora VM to validate the wipe story
- [ ] Document final setup as an implemented service page (per wiki lifecycle)

## Related

- [[docs/erebos/secrets]] — household age-key habits to reuse
- [[index]] — the vault itself: same "wipe-proof via git" philosophy

## External references

- **chezmoi** — docs: https://www.chezmoi.io · repo: https://github.com/twpayne/chezmoi · install: https://www.chezmoi.io/install/ · new-machine flow: https://www.chezmoi.io/user-guide/setup/
- **age** — repo: https://github.com/FiloSottile/age · chezmoi integration: https://www.chezmoi.io/user-guide/encryption/age/
- **agenix** (habits on Erebos) — repo: https://github.com/ryantm/agenix
- **home-manager** (alternative) — repo: https://github.com/nix-community/home-manager · docs: https://nix-community.github.io/home-manager/
- **Fedora packaging** — chezmoi in Fedora: https://packages.fedoraproject.org/pkgs/chezmoi/chezmoi/ · Flatpak: https://flatpak.org · dconf: https://gitlab.gnome.org/GNOME/dconf
