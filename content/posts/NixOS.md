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

Feel free to take a look at the configuration on [Github](https://github.com/TheWhale01/nixos-config)
> __*NOTE:*__ For now the github repo is in private since some commits contain secrets
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
├── shells/
│   └── ...
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
## Software development

I need to understand how software development works on NixOS. It is very simple ! In the `flake.nix` file you can declare different shells for different use. For example you can declare a shell with `nodejs` for web development:

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
		devShells.${system} = {
			node = (import ./shells/node.nix { inherit pkgs; });
		};
	};
}
```

And then you can begin to declare your environment:

```nix
# node.nix
{ pkgs, ... }:

pkgs.mkShell
{
	# Packages in the dev shell
	nativeBuildInputs = with pkgs; [
		nodejs
	];

	# Executed once the dev shell is started
	shellHook = ''
		echo 'webdev shell'
	'';
}
```

Here you can see that we just added nodejs you could add all the nixpkgs repo in there.

Finally, to enter you new dev environment just type this command:

```bash
nix develop .#node --command zsh
```

> __*NOTE:*__ the `--command zsh` part is not mandatory. By default NixOS will log you in a bash shell, since I'm using zsh as my main shell this ensures that the new dev shell will keep using my zsh configuration
## Erebos Configuration

I started by the server configuration since it's the core part of my workflow. I established a list of the services I'm using. Thanks you the incredibly large `nixpkgs` repo every services I'm currently using has a NixOS version. I dropped docker for most of the services so that the system takes up less RAM. I still enabled [Podman](https://podman.io/) for some services that wouldn't build, and maybe for development.

Everything is declarative appart from the [Tailscale](https://tailscale.com/) login. I did not find anything to properly get around that.

For this I'm using

 - [Home-Manager](https://nixos.wiki/wiki/Home_Manager) - Dotfiles configuration
 - [Disko](https://github.com/nix-community/disko) - Declarative disk partitioning
 - [Agenix](https://nixos.wiki/wiki/Agenix) - Secrets encryption

This configuration is made so that it can only be managed by Hades / root. So home-manager is manageable only by hades / root.

It is now at 90% done. I still need to package my blog stack for NixOS. Of course I could just declaratively recreate the containers but I think it's better to know the Nix way

**_UPDATE:_** Now the my blog pipeline is fully ported on NixOS. You can learn more [here](/content/posts/blog.md)