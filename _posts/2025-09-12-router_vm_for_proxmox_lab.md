---
layout: post
title: "Router VM for my ProxMox Lab"
date: 2025-09-12
author: "TimDude"
categories: ["Tech", "AD CS Lab"]
tags: ["proxmox", "linux", "routing"]
---

I have a dedicated Subnet on my ProxMox LAB so DHCP and DNS dont interfer with my home Network.
To be able to access machines on the dev Subnet of my Lab, i will set up a small router VM on ProxMox.


# Create an isolated bridge for the lab
On the ProxMox Box we will need to edit /etc/network/interfaces
```
auto vmbr1
iface vmbr1 inet manual
    bridge_ports none
    bridge_stp off
    bridge_fd 0
```
{: file="/etc/network/interfaces" }

# Spin up a small Linux Vm

## Assign interfaces

| interface | ip |
|-----|-----|
| eth0 | 192.168.178.5 |
| eth1 | 10.10.0.1 |

## Activate routing
Immediate activation
```
sysctl -w net.ipv4.ip_forward=1

```
Persisting the changes
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
```
