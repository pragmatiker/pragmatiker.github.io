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
Run reg as a container
```
podman run -d \
  -p 5000:5000 \
  --name registry \
  --restart=always \
  registry:2
```

verify
```
curl http://localhost:5000/v2/_catalog
```

## 3. Allow insecure local registry (important)
```
sudo nano /etc/containers/registries.conf.d/local.conf
```

```
[[registry]]
location = "localhost:5000"
insecure = true
```
{: file="/etc/containers/registries.conf.d/local.conf" }

## 4. Mirror Fedora bootc base image

Example:
```
skopeo copy \
  docker://quay.io/fedora/fedora-bootc:40 \
  docker://localhost:5000/fedora-bootc:40 \
  --dest-tls-verify=false
```
Verify:
```
curl http://localhost:5000/v2/_catalog
```


 





