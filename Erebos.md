---
title: Erebos
date: 2025-03-27
lastmod: 2025-03-27
author: whale
tags:
  - blog
  - nixos
  - VM
---
Like said in my [NixOS](NixOS.md) post [Erebos](Erebos.md) is the name of my server configuration. It's running every services I need and it's easily expandable. A true pleasure to work on.

Everything is declarative appart from the [Tailscale](https://tailscale.com/) login. I did not find anything to properly get around that.

For this I'm using

 - [Home-Manager](https://nixos.wiki/wiki/Home_Manager) - Dotfiles configuration
 - [Disko](https://github.com/nix-community/disko) - Declarative disk partitioning
 - [Agenix](https://nixos.wiki/wiki/Agenix) - Secrets encryption
 - [NixOS Modules](https://github.com/TheWhale01/nixos-modules) - Common configuration between all of my systems
 - [Blog Builder](https://github.com/TheWhale01/blog-builder) - Easily build and deploy an [Obsidian](https://obsidian.md/) vault as a static website.

This configuration is made so that it can only be managed by Hades / root. So home-manager is manageable only by hades / root.

It is now at 98% done ! I know for a fact that a nice 100% is not realistic. But I've got something more than usable. The only `critical` thing for now is the monitoring part. I've deployed grafana and prometheus but I've no alerting running and a poor log analyzer. Need to properly configure that !

## TODO:

 - Monitoring
 - Backups
 - Auto Podman image updates
