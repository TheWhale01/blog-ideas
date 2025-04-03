---
title: Fedora
date: 2025-03-19
lastmod: 2025-03-20
author: whale
---
I'm looking for a simple, configuration-less, stable, up-to-date distro to daily drive. Fedora looks like the perfect choice. I went from debian which have too old packages for me to archlinux which is rolling release fairly simple to use but can be tricky to maintain. Fedora also supports flatpak and hyprland which a great bonus for me. In addition I plan to buy a macbook pro m1 / m2 to install asahi linux on it (which is based on fedora) !

Today, after a weird behaviour of archlinux about sound. I decided to switch to Fedora ! Really easy to install (even too easy for me) even when dual booting to windows on a HP laptop. Once the installation done, I started setting up my environment. Good surprise, [Flatpak](https://flatpak.org/) is already installed !

I discovered that [GNOME](https://www.gnome.org/) is still using X11 sometimes, not really important as I wanted my [Hyprland](https://hyprland.org/) I started by installing it (really easy again) but the dotfiles that I used: [HyDE](https://github.com/Hyde-project/hyde) only work on arch. So I broke up the config files to have something portable (across distro) and also powerful. You can retrieve this project here: [https://github.com/TheWhale01/hyprland-config](https://github.com/TheWhale01/hyprland-config)

And a bit of tweaking here and there and I have something like the original experience:

## How did I make this portable ?

![Hyprland on fedora](hyprland_desktop.png)

**ALL** the configuration is located on `~/.config/hypr` using a simple bash script that creates symlinks `~/.config/hypr/app_conf/{app_name} -> ~/.config/{app_name}` I can have every configuration needed on one location (and accessible as usual) and backed-up on one single repository. Since I do not need (at least for now) all the features from [HyDE](https://github.com/Hyde-project/hyde) I can hard code / remove a lot of things from the original repo
## Improving

You probably noticed that there are not the same exact things in my [waybar](https://github.com/Alexays/Waybar) as [HyDE](https://github.com/Hyde-project/hyde) ! This is because they have a lot of bash script to automate things. Before I can port a feature I need to understand what the scripts does and how I can shrink it to perfectly match my needs. So there are these few things to keep:

- maybe setup a display manager
- setup polkit agent: [https://github.com/hyprwm/hyprpolkitagent](https://github.com/hyprwm/hyprpolkitagent)
- application theming (catppuccin mocha)
- enable systemd services (Now using exec-once directive from hyprland)

I also found this github repo [https://github.com/gaurav23b/simple-hyprland](https://github.com/gaurav23b/simple-hyprland) that is explaining all the process of ricing hyprland can be very useful.