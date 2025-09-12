---
layout: post
title: "Router VM for my ProxMox Lab"
date: 2025-07-25 10:00:00 -0000
author: "TimDude"
categories: ["Tech",  "AD CS Lab"]
tags: ["proxmox", "linux", "routing"]
---

I have a dedicated Subnet on my ProxMox LAB so DHCP and DNS dont interfer with my home Network.
To be able to access machines on the dev Subnet of my Lab, i will set up a small router VM on ProxMox.



On the ProxMox Box we will need to add a Bridge
```
auto vmbr1
iface vmbr1 inet manual
    bridge_ports none
    bridge_stp off
    bridge_fd 0
```
{: file="/etc/network/interfaces" }
