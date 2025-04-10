---
title: Remote Gaming Server
date: 2025-03-22
lastmod: 2025-03-22
author: whale
---
This idea seems pretty simple at first, self host a cloud gaming service. The simplest option would be to get steam remote play to work on a headless server. Looking for something existing on internet, it seems that it's not really possible. So maybe a [Sunshine](https://github.com/LizardByte/Sunshine) server would be the best option. I first need to buy a graphics card I'm thinking about a [RX 7600XT](https://www.amd.com/fr/products/graphics/desktops/radeon/7000-series/amd-radeon-rx-7600-xt.html#tabs-05d56812ab-item-412fac163c-tab) for its great compatibility / performance on Linux.

I've tried Moonlight but it needs a display connected, it can be a virtual display driver or a dummy HDMI. At the moment I'm writing this blog post I don't have a physical access to my server so I can't plug a dummy HDMI into the graphics card. As of the virtual display driver it seems overly complicated and I don't want to mess around with stuff I don't understand. So I came across [Apollo](https://github.com/ClassicOldSong/Apollo) which is a fork of [Sunshine](https://github.com/LizardByte/Sunshine) which is (for now) mainly supported on windows and supports Virtual Displays. After several attempts and a long time reverse engineering the build process I was able to build [Apollo](https://github.com/ClassicOldSong/Apollo) on Linux. But as said in the `README` these are only available on Windows, for now...