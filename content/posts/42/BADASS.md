---
title: BADASS
date: 28-10-2025
lastmod: 28-10-2025
author: whale
tags:
  - school
  - cloud
  - docker
  - linux
  - VM
---

BADASS (For Bgp At Doors of Autonomous System is Simple) is a project that is about networking. In this we will have to setup multiple networks implementing different routing protocols and viewing / debugging them using different tools.

For more informations here is the Subject:
[https://cdn.intra.42.fr/pdf/pdf/160212/en.subject.pdf](https://cdn.intra.42.fr/pdf/pdf/160212/en.subject.pdf)

## Introduction

There are many new notions to learn which need to be understood to begin the work on this project.

**FR Routing:** _Free Range Routing_ -> This is implementing protocols using multiple routing procotols like **BGP**, **OSPF**, **IS-IS** and others to connect devices between them. We will discover these protocols during this project.

**BGP:** Considered like the _backbone of the internet_ it's a protocol that dictates how two networks should be connected (and so, how they should discuss). This is an old protocol, as so as, it supports **ONLY** IPv4.

**GNS3:** Program that lets you emulate / debug virtual networks

**Wireshark:** Tool to see packet sent through a connected network

**MP-BGP:** This one of the newer implementation of the BGP protocol that supports multiple other communication type (**IPv6**, **VPN**, ...)

**Busybox:** Is a condensed executable that includes multiple essential CLI tools. It comes in handy when working with a limited system both in terms of storage and performance.

**OSPF:** Protocol to determine the shortest path that a packet sent through a network should take. This allows to have minimum response time (PING) when working with large scale networks
