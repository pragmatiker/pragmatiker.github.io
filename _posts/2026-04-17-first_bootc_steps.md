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





## Base system prep

```
sudo apt update
sudo apt install -y podman skopeo curl
```


## Start local registry
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

## Allow insecure local registry (important)
```
sudo nano /etc/containers/registries.conf.d/local.conf
```

```
[[registry]]
location = "localhost:5000"
insecure = true
```
{: file="/etc/containers/registries.conf.d/local.conf" }

## Mirror Fedora bootc base image

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

## Build your own bootc image (important)

Using the Fedora base directly works, but it won’t auto-update because the tag is static.
So we create our own image that we control.

Create a Containerfile
```
FROM localhost:5000/fedora-bootc:40

# Make system quieter on serial console
RUN mkdir -p /usr/lib/bootc/kargs.d && \
    echo "quiet loglevel=3" > /usr/lib/bootc/kargs.d/quiet.conf

# Add some basic tools
RUN dnf install -y vim htop tmux && dnf clean all
```
{: file="./Containerfile" }


Build and tag image
```
podman build -t localhost:5000/my-bootc:1 .
podman tag localhost:5000/my-bootc:1 localhost:5000/my-bootc:latest
```

Push to local registry
```
podman push --tls-verify=false localhost:5000/my-bootc:1
podman push --tls-verify=false localhost:5000/my-bootc:latest
```

Verify
```
curl http://localhost:5000/v2/_catalog
curl http://localhost:5000/v2/my-bootc/tags/list
```


 ## Prepare build workspace
 ```
mkdir -p ~/bootc-build/output
cd ~/bootc-build
```

## (Optional but recommended) Add user config
```
[[customizations.user]]
name = "therty"
password = "$6$QuzSFM....3zohPb0MypZ/"
groups = ["wheel"]

[[customizations.sshkey]]
user = "therty"
key = "ssh-ed25519 AAAAC3NzaC....k6eqHy4ll1xnhhJGbR"
```
{: file="./config.toml" }

Generate password hash (example):
```
openssl passwd -6
```

## Build qcow2 image
```
sudo podman pull localhost:5000/my-bootc:latest

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
  localhost:5000/my-bootc:latest
```

Output:
```
./output/qcow2/disk.qcow2
```

## Copy to proxmox
```
scp output/qcow2/disk.qcow2 root@192.168.100.1:/root
```
## Create VM from Image
On the Proxmox Host import the image and create Vm from it.

```
qm create 9000 --name fedora-bootc --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
qm importdisk 9000 disk.qcow2 local-lvm
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --boot order=scsi0
qm set 9000 --serial0 socket --vga serial0
```



