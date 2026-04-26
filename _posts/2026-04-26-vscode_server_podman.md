---
layout: post
title: "VSCode Server in Podman"
date: 2026-04-26
author: "TimDude"
categories: ["Tech", "devops"]
tags: ["podman", "vscode"]
---

So lets run VSCode in a Webbrowser via VSCode Serer running in a conatiner on podman


## Goals
Automate container builds:
- Run VSCode in Container
- Have persistent storage
-  Have it start via systemd

## Container Setup

### Install podman
On debian
```
apt update
apt install podman -y
```

### Create VSCode Server Container

Create folders for persistent Storage
```
mkdir -p ~/code-server/{project,config,local}
```

```
podman run -d \
  --name code-server \
  -p 8080:8080 \
  --userns=keep-id \
  -v "$HOME/code-server/project:/home/coder/project:Z" \
  -v "$HOME/code-server/config:/home/coder/.config:Z" \
  -v "$HOME/code-server/local:/home/coder/.local:Z" \
  docker.io/codercom/code-server:latest \
  --bind-addr 0.0.0.0:8080
```

### Create systemd file

```
podman generate systemd --name code-server --files --new
```

creates:
```
container-code-server.service
```

### Activate the systemd Unit
```
mkdir -p ~/.config/systemd/user/
mv container-code-server.service ~/.config/systemd/user/

systemctl --user daemon-reload
systemctl --user enable --now container-code-server
```

Let the Unit start without logging into the account
```
sudo loginctl enable-linger $USER
```
