---
layout: post
title: "First steps with bootc"
date: 2026-04-17
author: "TimDude"
categories: ["Tech", "containers"]
tags: ["proxmox", "bootc", "podman"]
---

So lets experient with bootable containers.

At a high level: we are not installing a system — we are building an image and booting it.
# Goals:
* Boot a bootc Linux in proxmox
* Edit the container
* bootc updgrade


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
curl http://192.168.100.10:5000/v2/_catalog
```

## Allow insecure local registry (important)
```
sudo nano /etc/containers/registries.conf.d/local.conf
```

```
[[registry]]
location = "192.168.100.10:5000"
insecure = true
```
{: file="/etc/containers/registries.conf.d/local.conf" }

## Mirror Fedora bootc base image

Example:
```
skopeo copy \
  docker://quay.io/fedora/fedora-bootc:40 \
  docker://192.168.100.10:5000/fedora-bootc:40 \
  --dest-tls-verify=false
```
Verify:
```
curl http://192.168.100.10:5000/v2/_catalog
```

## Prepare build workspace

```
mkdir -p ~/bootc-build/output
cd ~/bootc-build
```

## Build your own bootc image (important)

Using the Fedora base directly works, but it won’t auto-update in our setup,
because we mirror it into a local registry and the tag stays unchanged.

Create a Containerfile
```
FROM 192.168.100.10:5000/fedora-bootc:40

RUN mkdir -p /etc/containers/registries.conf.d && \
    cat > /etc/containers/registries.conf.d/local.conf <<'EOF'
[[registry]]
location = "192.168.100.10:5000"
insecure = true
EOF

# Correct bootc kargs.d format: TOML
RUN cat > /usr/lib/bootc/kargs.d/10-console.toml <<'EOF'
kargs = ["quiet", "loglevel=3"]
EOF

# Lower runtime console verbosity too
RUN cat > /usr/lib/sysctl.d/10-kernel-printk.conf <<'EOF'
kernel.printk = 3 4 1 3
EOF

# Add some basic tools. We save this for v3
#RUN dnf install -y vim htop tmux && \
#    dnf clean all
```
{: file="./Containerfile" }


Build and tag image

We use two tags:
- a fixed version (`:1`) for reproducibility
- a moving tag (`:latest`) for automatic upgrades

```
podman build -t 192.168.100.10:5000/my-bootc:1 .
podman tag 192.168.100.10:5000/my-bootc:1 192.168.100.10:5000/my-bootc:latest
```

Push to local registry
```
podman push --tls-verify=false 192.168.100.10:5000/my-bootc:1
podman push --tls-verify=false 192.168.100.10:5000/my-bootc:latest
```

Verify
```
curl http://192.168.100.10:5000/v2/_catalog
curl http://192.168.100.10:5000/v2/my-bootc/tags/list
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
sudo podman pull 192.168.100.10:5000/my-bootc:latest

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
  192.168.100.10:5000/my-bootc:latest
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



