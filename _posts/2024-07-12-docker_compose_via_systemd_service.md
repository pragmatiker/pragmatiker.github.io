---
layout: post
title: "Docker Compose via Systemd Service"
date: 2024-07-12 12:00:00 -0000
author: "TimDude"
categories: ssh_poc docker
tags: systemd docker docker-compose
---

All containers of the SSH CA proof of concept lab will end up beeing started via the same 
docker-compose.yml under /opt/docker/ssh_poc

## Let our Lab start at system boot
Add systemd Service /etc/systemd/system/docker-compose-ssh_poc.service
~~~
[Unit]
Description=Docker Compose Application (ssh_poc)
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=true
WorkingDirectory=/opt/docker/ssh_poc
ExecStart=/usr/bin/docker-compose up -d
ExecStop=/usr/bin/docker-compose down

[Install]
WantedBy=multi-user.target
~~~

Enable the Service
~~~
systemctl enable docker-compose-ssh_poc.service
~~~
