---
layout: post
title: "1. Home lab lan Structure"
date: 2026-05-03
author: "TimDude"
categories: ["Tech","Home lab"]
tags: ["proxmomx", "pki", "dns", "podman", "blue-build", "gitlab"]
---


Home Lab

| Name | IP | Function |
|-------|--------|---------|
| pve | 192.168.100.1 | Router / Gateway |
| dns | 192.168.100.2 | Bind9 DNS Server |
| ca | 192.168.100.3 | smallstep TLS CA |
| registry | 192.168.100.10 | Conatainer registry |
| gitlab | 192.168.100.11 | gitlab |
