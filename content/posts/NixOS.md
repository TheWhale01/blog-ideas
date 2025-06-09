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
- erebos means hell in ancient greek and its user is hades
- pontos means sea in ancient greek and its user is poseidon
- olympos means olympus in ancient greek and its user is zeus.

## Installation

So, to make a nixos configuration you need, NixOS ! So I've made a virtual machine and the installation can be done through a GUI or a command line. Nothing changes just take whatever you like. You can follow the installtion guide there: [https://nixos.wiki/wiki/NixOS_Installation_Guide](https://nixos.wiki/wiki/NixOS_Installation_Guide)