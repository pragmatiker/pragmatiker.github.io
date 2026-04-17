---
layout: post
title: "First steps with bootc"
date: 2026-04-17
author: "TimDude"
categories: ["Tech", "containers"]
tags: ["proxmox", "bootc", "podman"]
---

So lets experient with bootable containers.
# Goals:
* Boot a bootc Linux in proxmox
* Edit the container
* bootc Updgrade





## Configure reg
```
ctrl_interface=/run/wpa_supplicant
update_config=1
country=DE

network={
    ssid="YOUR_WIFI_NAME"
    psk="YOUR_PASSWORD"
}
```
{: file="/etc/wpa_supplicant/wpa_supplicant.conf" }


 





