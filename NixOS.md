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
I wanted to switch to NixOS not only for its immutable kernel but for its declarative configuration. This allows to backup these configurations to automatically setup your particular system on a brand new computer. For example you could turn an old laptop into a server simply by cloning the configuration and deploying it. So I wanted to setup three different configurations:

- [erebos](Erebos.md) - server
- pontos - laptop
- olympos - desktop

The names come from the greek mythology:

- `erebos` means `hell` in ancient greek and its user is Hades
- `pontos` means `sea` in ancient greek and its user is Poseidon
- `olympos` means `olympus` in ancient greek and its user is Zeus
- `strategos` means `strategy` in ancient greek and its user is Athena

Feel free to take a look at the configuration on [Github](https://github.com/TheWhale01/nixos-config)

> __*NOTE:*__ For now the github repo is in private since some commits contain secrets
## Installation

So, to make a NixOS configuration you need, NixOS ! So I've made a virtual machine and the installation can be done through a GUI or a command line. Nothing changes just take whatever you like. You can follow the installtion guide there: [https://nixos.wiki/wiki/NixOS_Installation_Guide](https://nixos.wiki/wiki/NixOS_Installation_Guide)

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
│── configuration.nix
│── disko.nix
│── hardware-configuration.nix
│── home.nix
│── secrets.nix
├── readme.md
│── home-manager/
│   └── ...
│   sys/
│   ├── containers/
│   │   └── ...
│   ├── programs/
│   │   └── ...
│   ├── services/
│   │   └── ...
│   └── vars.nix
├── modules/
│   └── ...
└── secrets/
    └── ...
```

## Software development
One benefit of NixOS is to declaratively describe development environments. Like everything else in NixOS there are two main ways to do this. `Flakes` and `Channels`. Since I'm using flakes for my system configuration I will also use them for my projects. Environments are declared in a `flake.nix` file. This file define not only the development environment but also how to compile and deploy your project under NixOS.

```nix
# flake.nix
{
	inputs = {
		nixpkgs.url = "github:nixos/nixpkgs/release-25.05";
		# Other system dependencies...
	};
	outputs = { nixpkgs, ... }@inputs:
	let
		system = "x86_64-linux";
		lib = nixpkgs.lib;
		pkgs = import nixpkgs {
			system = "${system}";
		};
	in {
		devShells.${system}.default = {
            # Your default dev env
		};
        devShells.${system}.node = {
            # Your node dev env
        };
	};
}
```

Here you can see that we just added nodejs you could add all the nixpkgs repo in there.

Finally, to enter you new dev environment just type this command:

```bash
nix develop .
# or for a particular env
nix develop .#node
```

For ease of use you can also use [direnv](https://wiki.nixos.org/wiki/Direnv). This will automatically enable a dev environment if declared if a `.envrc` file is present in your project directory.

```bash
echo 'use flake' > .envrc
direnv allow .
```