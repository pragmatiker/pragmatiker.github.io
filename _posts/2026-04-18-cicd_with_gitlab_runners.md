---
layout: post
title: "CI/CD via GitLab Runners"
date: 2026-04-17
author: "TimDude"
categories: ["Tech", "devops"]
tags: ["gitlab", "ci-cd"]
---

So lets set up a Pipeline for buidling containers


## Goals
Automate container builds:

Containerfile change
→ GitLab pipeline triggers
→ Runner builds image via Podman
→ Image tagged (incrementing + latest)
→ Image pushed to local registry

## Architecture

GitLab (server)
    ↓
GitLab Runner (build host, shell executor)
    ↓
Podman (build + push)
    ↓
Local registry (192.168.100.10:5000)


## Runner Setup

### Install
```
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash sudo apt install -y gitlab-runner podman
```

