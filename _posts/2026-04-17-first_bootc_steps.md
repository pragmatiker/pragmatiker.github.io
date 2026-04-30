---
layout: post
title: "First steps with bootc"
date: 2026-04-18
author: "TimDude"
categories: ["Tech", "devops"]
tags: ["proxmox", "bootc", "podman", "containers"]
---

So lets experiment with bootable containers.

At a high level: we are not installing a system — we are building an image and booting it.
## Goals
- Boot a bootc-based Fedora system in Proxmox
- Customize the OS by building a derived container image
- Upgrade the machine by publishing a new image and applying it with bootc


## Registry with webUI
### Base system prep

```
sudo apt update
sudo apt install -y podman skopeo curl
```

### Make data dir
```
sudo mkdir -p /srv/registry
```

### create config

```
mkdir -p /srv/registry /srv/registry-config

cat > /srv/registry-config/config.yml <<'EOF'
version: 0.1

log:
  fields:
    service: registry

storage:
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true

http:
  addr: :5000
  headers:
    Access-Control-Allow-Origin: ['http://192.168.100.10:8080']
    Access-Control-Allow-Methods: ['HEAD', 'GET', 'OPTIONS', 'DELETE']
    Access-Control-Allow-Headers: ['Authorization', 'Accept', 'Cache-Control', 'Content-Type']
    Access-Control-Expose-Headers: ['Docker-Content-Digest']
EOF
```

### Start local registry
Run reg as a container
```
podman run -d --replace \
  --name registry \
  -p 5000:5000 \
  -v /srv/registry:/var/lib/registry:Z \
  -v /srv/registry-config:/etc/docker/registry:Z,ro \
  docker.io/library/registry:2
```
Run webUI as container
```
podman run -d --replace \
  --name registry-ui \
  --restart=always \
  -p 8080:80 \
  -e REGISTRY_TITLE="My Registry" \
  -e REGISTRY_URL="http://192.168.100.10:5000" \
  -e DELETE_IMAGES=true \
  docker.io/joxit/docker-registry-ui:main-debian
```


### Create systemd services
Remember reg it is run from systemd as root.
So normal user "podman ps" wont show it

```
podman generate systemd --name registry --files --new

sudo mv container-registry.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now container-registry.service
sudo podman ps -a
```

```
podman generate systemd --name registry-ui --files --new
sudo mv container-registry-ui.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now container-registry-ui.service
sudo podman ps -a
```

verify
```
curl http://192.168.100.10:5000/v2/_catalog
```

## Build container

### Allow insecure local registry (important)

```
sudo nano /etc/containers/registries.conf.d/local.conf
```

```
[[registry]]
location = "192.168.100.10:5000"
insecure = true
```
{: file="/etc/containers/registries.conf.d/local.conf" }

### Mirror Fedora bootc base image

Example:
```
skopeo copy \
  docker://quay.io/fedora/fedora-kinoite:42 \
  docker://192.168.100.10:5000/kinoite:42 \
  --dest-tls-verify=false
```
Verify:
```
curl http://192.168.100.10:5000/v2/_catalog
```

### Prepare build workspace

```
mkdir -p ~/bootc-build/output
cd ~/bootc-build
```

### Build your own bootc image (important)

Using the Fedora base directly works, but it won’t auto-update in our setup,
because we mirror it into a local registry and the tag stays unchanged.

Create a Containerfile
```
FROM 192.168.100.10:5000/kinoite:42

### Version 1 ####
# Connect to unsafe regs
RUN mkdir -p /etc/containers/registries.conf.d
RUN cat > /etc/containers/registries.conf.d/local.conf <<'EOF'
[[registry]]
location = "192.168.100.10:5000"
insecure = true
EOF

# Correct bootc kargs.d format: TOML
#RUN mkdir -p /usr/lib/bootc/kargs.d
#RUN cat > /usr/lib/bootc/kargs.d/10-console.toml <<'EOF'
#kargs = ["quiet", "loglevel=3"]
#EOF

# Lower runtime console verbosity too
#RUN mkdir -p /usr/lib/sysctl.d
#RUN cat > /usr/lib/sysctl.d/10-kernel-printk.conf <<'EOF'
#kernel.printk = 3 4 1 3
#EOF
### Version 1 END ###
```
{: file="./Containerfile" }


Build and tag image

We use two tags:
- a fixed version (`:42`) for reproducibility
- a moving tag (`:stable`) for automatic upgrades

```
podman build -t 192.168.100.10:5000/kinoite-base:42-v1 .
podman tag 192.168.100.10:5000/kinoite-base:42 192.168.100.10:5000/kinoite-base:stable
```

Push to local registry
```
podman push --tls-verify=false 192.168.100.10:5000/kinoite-base:42-v1
podman push --tls-verify=false 192.168.100.10:5000/kinoite-base:stable
```

Verify
```
curl http://192.168.100.10:5000/v2/_catalog
curl http://192.168.100.10:5000/v2/kinoite-base/tags/list
```

## Build DiskImage for Proxmox


### Switch to build dir
```
cd ~/bootc-build
```

### (Optional but recommended) Add user config
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

### Build qcow2 image
```
sudo podman --tls-verify=false pull 192.168.100.10:5000/kinoite-base:stable

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
  192.168.100.10:5000/kinoite-base:stable
```

Output:
```
./output/qcow2/disk.qcow2
```


## Create VM from Disk Image

### Copy to proxmox
```
scp output/qcow2/disk.qcow2 root@192.168.100.1:/root
```

### Create VM from Image

On the Proxmox host, import the image and create a VM from it.

```
qm create 9000 --name kinoite-base --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
qm importdisk 9000 disk.qcow2 local-lvm
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --boot order=scsi0
qm set 9000 --serial0 socket --vga serial0
```

### Networking
Just in case you have no dhcp
```
sudo nmcli connection add type ethernet ifname ens18 con-name ens18 \
  ip4 192.168.100.20/24 gw4 192.168.100.1

sudo nmcli connection modify ens18 ipv4.dns "1.1.1.1"
sudo nmcli connection up ens18
```

## Updating the system with bootc
Now the fun part: the host OS is updated from a new container image.

You generally do not mutate the host with ad-hoc package installs.
Instead, you change the Containerfile, build a new image, push it to the registry,
and then let the machine apply that image with `bootc upgrade`.

### Build new Container v2
Create a Containerfile

In practice, you would keep a single Containerfile and evolve it.
We split it here into "Version 1" and "Version 2" for clarity.
```
FROM 192.168.100.10:5000/kinoite-base:42

### Version 1 ####
# Connect to unsafe regs
RUN mkdir -p /etc/containers/registries.conf.d
RUN cat > /etc/containers/registries.conf.d/local.conf <<'EOF'
[[registry]]
location = "192.168.100.10:5000"
insecure = true
EOF

# Correct bootc kargs.d format: TOML
#RUN mkdir -p /usr/lib/bootc/kargs.d
#RUN cat > /usr/lib/bootc/kargs.d/10-console.toml <<'EOF'
#kargs = ["quiet", "loglevel=3"]
#EOF

# Lower runtime console verbosity too
#RUN mkdir -p /usr/lib/sysctl.d
#RUN cat > /usr/lib/sysctl.d/10-kernel-printk.conf <<'EOF'
#kernel.printk = 3 4 1 3
#EOF
### Version 1 END ###

### Version 2 ###
# Add some basic tools.
RUN dnf install -y htop
RUN dnf clean all
### Version 2 END ###
```
{: file="./Containerfile" }

Build and tag image
Like before, we use 2 tags: one fixed version and `stable`.

```
podman build -t 192.168.100.10:5000/kinoite-base:42-v2 .
podman tag 192.168.100.10:5000/kinoite-base:42-v1 192.168.100.10:5000/kinoite-base:stable
```

Push to local registry
```
podman push --tls-verify=false 192.168.100.10:5000/kinoite-base:42-v2
podman push --tls-verify=false 192.168.100.10:5000/kinoite-base:stable
```

### Apply the update on the VM

Check for a new image:
```
sudo bootc upgrade
```

Apply it
```
sudo bootc upgrade --apply
```

reboot
```
sudo reboot
```


