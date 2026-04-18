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
```
GitLab (server)
    ↓
GitLab Runner (build host, shell executor)
    ↓
Podman (build + push)
    ↓
Local registry (192.168.100.10:5000)
```


## Runner Setup

### Install
Add repo
```
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
```
install
```
sudo apt install -y gitlab-runner podman
```

### Register
```
sudo gitlab-runner register
```

Settings:

URL: http://192.168.100.10 # Gitlab URL
Executor: shell
Tags: podman
Description: build-host

## Podman Registry Config (on runner)

File:
```
[[registry]]
location = "192.168.100.10:5000"
insecure = true
```
{: file="/etc/containers/registries.conf.d/local.conf" }


## Gitlab Project

### Project Structure
Containerfile
.gitlab-ci.yml

### GitLab CI Pipeline
```
stages:
  - build

build_container:
  stage: build
  tags:
    - podman
  variables:
    IMAGE: "192.168.100.10:5000/my-bootc"
  script:
    - BUILD_TAG="$CI_PIPELINE_IID"

    - echo "Building $IMAGE:$BUILD_TAG"
    - podman build -t "$IMAGE:$BUILD_TAG" .

    - echo "Tagging latest"
    - podman tag "$IMAGE:$BUILD_TAG" "$IMAGE:latest"

    - echo "Pushing numbered tag"
    - podman push --tls-verify=false "$IMAGE:$BUILD_TAG"

    - echo "Pushing latest tag"
    - podman push --tls-verify=false "$IMAGE:latest"
  rules:
    - changes:
        - Containerfile
```
{: file=".gitlab-ci.yaml" }

### Tagging Strategy
Incrementing: $CI_PIPELINE_IID
Rolling: latest

Example:

my-bootc:12
my-bootc:latest

### Execution Flow
1. Commit changes to Containerfile
2. Push to GitLab
3. Pipeline triggers
4. Runner executes:
5. podman build
6. podman tag
7. podman push
8. Image available in registry


