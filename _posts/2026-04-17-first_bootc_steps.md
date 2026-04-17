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





## 1. Base system prep

```
sudo apt update
sudo apt install -y podman skopeo curl
```


## 2. Start local registry
```
podman run -d \
  -p 5000:5000 \
  --name registry \
  --restart=always \
  registry:2
```



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


 





