---
title: Whale's Server
date: 2025-03-27
lastmod: 2025-03-27
author: whale
---
I'm very happy with how my server is running, for now its powerful enough for my use case (even missing a GPU unfortunately), but as I said in my [Fedora](/content/posts/Fedora.md) post I want a configuration-less, stable, and up-to-date distribution. Fedora is fine but if I need to change the distro or the hard disk I will need to recreate from scratch the configuration or to save every service specific configuration on a GitHub repo. So [NixOS]() seems the best distribution because it is fully declarative that makes it very easily reproducible and I think it is particularly good for a server use thank to its immutable kernel, forcing the use of virtualization / containerization / sandboxing, potentially increasing security.

But I don't know well this distro it seems that developping can be a little more complicated but once configured properly the program is can be shipped to any Linux ditro. I don't know either if all the dotfiles (regardless of the application) can be managed at once. It seems like yes using [Home-Manager](https://nixos.wiki/wiki/Home_Manager).

I would like to run it on my actual Debian server but with openGL support (seems impossible with virt-manager: [https://askubuntu.com/questions/1348975/how-to-use-opengl-3d-acceleration-in-virt-manager-with-ubuntu](https://askubuntu.com/questions/1348975/how-to-use-opengl-3d-acceleration-in-virt-manager-with-ubuntu))
