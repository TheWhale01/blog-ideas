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
I've been digging around this subject for a few days at the time I'm writting this blog post. And NixOS is an absolute beast ! Depending on some simple configuration files you can go from a server oriented OS to a gaming like experience just by copying / pasting the right files.

## NixOS ISO Installation

First of all, like any other OS you have to create a bootable usb stick. On this page you can find some NixOS flavours depending on what you want to do 

> __*NOTE:*__ As I said previously, every single configuration is explicitly specified in some files located in `/etc/nixos` so you could easily go from one flavour to another

[https://nixos.wiki/wiki/NixOS_Installation_Guide](https://nixos.wiki/wiki/NixOS_Installation_Guide)

Personally I did choose the minimal installation so that I can build everything the way I want. Formatting the hard drive to prepare the installation, adding a user, and create a root password, reboot (I let you follow the nix wiki very simple) and here comes the fun part: **configuration!**

## NixOS Configuration

the main configuration the main configuration file is located under `/etc/nixos/configuration.nix`

I'm also going for a daily driver configuration based on my [hyprland config](/content/posts/Fedora.md). So I need to extract all the dependencies to make my configuration work and place it in the `configuration.nix` file as follow:

```nix
# /etc/configuration.nix
environment.systemPackages = with pkgs; [
	# Specify the name of the packages you want to install
	# ex:
	# vim
	# wget
	# kitty
];
```

I've also using home-manager to make any app configuration (dotfiles) for any users. This part is very simple at the first sight : make the `.config` directory declarative. But it actually is very very deep, with a lots of ways to properly install home-manager. The best option I've found are some videos on youtube and the official documentation.

Learning about home-manager I came across the _flake_ way of doing things. This is I think the best option because it lets you specify any version number in the configuration and update them if needed and create a git repository to push your configuration. And it even lets you describe multiple configurations. About home manager, I want every user to be able to customize their home-manager configuration. For the example here is my flake.nix file:

```nix
# flake.nix
{
	description = "NixOS system configuration";

	inputs = {
		nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
		home-manager.url = "github:nix-community/home-manager";
	};

	outputs = {self, nixpkgs, home-manager, ... }:
		let
			system = "x86_64-linux";
			lib = nixpkgs.lib;
			pkgs = nixpkgs.legacyPackages.${system};
		in {
		nixosConfigurations = {
			nix-whale = lib.nixosSystem {
				inherit system;
				modules = [
					./configuration.nix
				];
			};

		};
		homeConfigurations = {
			whale = home-manager.lib.homeManagerConfiguration {
				inherit pkgs;
				modules = [
					./home-manager/home.nix
				];
			};
		};
	};
}
```

I could, for example, put the home `home-manager` module in the `nixosSystem` function to have something managable **ONLY** by root.