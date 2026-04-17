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


 ## 5. Prepare build workspace
 ```
mkdir -p ~/bootc-build/output
cd ~/bootc-build
```

## 6. (Optional but recommended) Add user config
```
[[customizations.user]]
name = "therty"
password = "$6$QuzSFM....3zohPb0MypZ/"
groups = ["wheel"]

[[customizations.sshkey]]
user = "therty"
key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5Lfs2NpChZkyaVfG+ZZLcf+f7Lk6eqHy4ll1xnhhJGbR"
```
{: file="./config.toml" }

Generate password hash (example):
```
openssl passwd -6
```

## 7. Build qcow2 image
```
sudo podman pull localhost:5000/fedora-bootc:40

sudo podman run --rm -it \
  --privileged \
  --pull=newer \
  --security-opt label=type:unconfined_t \
  -v "$PWD/output:/output" \
  -v "$PWD/config.toml:/config.toml:ro" \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  quay.io/centos-bootc/bootc-image-builder:latest \
  --type qcow2 \
  --rootfs btrfs \
  --config /config.toml \
  localhost:5000/fedora-bootc:40
```

Output:
```
./output/qcow2/disk.qcow2
```






