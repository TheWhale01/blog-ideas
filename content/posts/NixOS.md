---
title: NixOS
date: 2025-04-18
lastmod: 2025-04-18
author: whale
tags:
  - blog
  - nixos
  - linux
  - server
---
I wanted to switch to NixOS not only for its immutable kernel but for its declarative configuration. This allows to backup these configurations to automatically setup your particular system on a brand new computer. For example you could turn an old laptop into a server simply by cloning the configuration and running it. So I wanted to setup three different configurations:

- erebos - server
- pontos - laptop
- olympos - desktop

The names come from the greek mythology:
- `erebos` means `hell` in ancient greek and its user is Hades
- `pontos` means `sea` in ancient greek and its user is Poseidon
- `olympos` means `olympus` in ancient greek and its user is Zeus

## Installation

So, to make a nixos configuration you need, NixOS ! So I've made a virtual machine and the installation can be done through a GUI or a command line. Nothing changes just take whatever you like. You can follow the installtion guide there: [https://nixos.wiki/wiki/NixOS_Installation_Guide](https://nixos.wiki/wiki/NixOS_Installation_Guide)

To install one of my configurations:
```bash
# Make disk partitioning. Some hosts do not have disko enabled
sudo nix --experiemntal-features "nix-command flakes" run github:nix-community/disko -- --mode disko ./hosts/<erebos|pontos|olympos>/disko.nix

# Install system
sudo nixos-install --flake .#<erebos|pontos|olympos>
```

> __*NOTE:*__ I'm using nix flakes to backup the configuration on gihtub

Here is the directory structure:
```
nix/
├── flake.lock
├── flake.nix
├── hosts/
│   ├── erebos/
│   │   ├── configuration.nix
│   │   ├── disko.nix
│   │   ├── hardware-configuration.nix
│   │   ├── home-manager/
│   │   │   └── ...
│   │   ├── home.nix
│   │   ├── secrets.nix
│   │   └── system/
│   │       ├── containers/
│   │       │   └── ...
│   │       ├── programs/
│   │       │   └── ...
│   │       ├── services/
│   │       │   └── ...
│   │       └── vars.nix
│   └── pontos/
│       ├── configuration.nix
│       ├── hardware-configuration.nix
│       ├── home-manager/
│       │   └── ...
│       ├── home.nix
│       ├── packages.nix
│       └── stylix.nix
├── modules/
│   └── ...
├── readme.md
└── secrets/
    └── ...
```
## Erebos Configuration

I started by the server configuration since it's the core part of my workflow. I established a list of the services I'm using. Thanks you the incredibly large `nixpkgs` repo every services I'm currently using has a NixOS version. I dropped docker for most of the services so that the system takes up less RAM. I still enabled [Podman](https://podman.io/) for some services that wouldn't build, and maybe for development.

Everything is declarative appart from the [Tailscale](https://tailscale.com/) login. I did not find anything to properly get around that.

For this I'm using

 - [Home-Manager](https://nixos.wiki/wiki/Home_Manager) - Dotfiles configuration
 - [Disko](https://github.com/nix-community/disko) - Declarative disk partitioining
 - [Agenix](https://nixos.wiki/wiki/Agenix) - Secrets encryption

The goal of this configuration is to have the most services accessible system wide so that I can create other users and assign them permissions on these services.