---
layout: post
title: "Set up Gitlab"
date: 2026-04-17
author: "TimDude"
categories: ["Tech", "devops"]
tags: ["gitlab", "ci-cd"]
---

So lets set up gitlab


## Goals
- Installed gitlab on debian
- WebUI up and running

## VM
### VM sizing

For your Proxmox:

CPU: 2 vCPU (4 if you can spare it)
RAM: 6–8 GB
Disk: 40–60 GB
OS: Debian 12 minimal

## Gitlab install

### Install dependencies

```
sudo apt update
sudo apt install -y curl ca-certificates openssh-server tzdata perl
```
### Add GitLab repository
```
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
```

### Install GitLab

Set your URL first (important):
```
sudo EXTERNAL_URL="http://192.168.100.11" apt install gitlab-ce
```


```
[[registry]]
location = "192.168.100.10:5000"
insecure = true
```
{: file="/etc/containers/registries.conf.d/local.conf" }

