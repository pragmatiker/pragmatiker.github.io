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
GitLab (server 192.168.100.11)
    ↓
GitLab Runner (build host with shell executor 192.168.100.10:5000)
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

### add subid

```
sudo echo 'gitlab-runner:100000:65536' >> /etc/subuid
sudo echo 'gitlab-runner:100000:65536' >> /etc/subgid
```

### Create Accesstoken in gitlab

Go to:

Project → Settings → CI/CD → Runners

Click:

New project runner

Copy:

--token XXXXX


### Register
```
sudo gitlab-runner register --token XXXXX
```

Settings:

```
URL: http://192.168.100.11 # Gitlab URL
Executor: shell
Tags: podman
Description: build-host
```

### Podman Registry Config (on runner)

File:
```
[[registry]]
location = "192.168.100.10:5000"
insecure = true
```
{: file="/etc/containers/registries.conf.d/local.conf" }

### Test
```
sudo -iu gitlab-runner
podman pull --tls-verify=false 192.168.100.10:5000/fedora-bootc:40
```


## Gitlab Project

### Project Structure
Containerfile
.gitlab-ci.yml

### GitLab CI Pipeline

My registry is running at: http://192.168.100.10:5000
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
    - BUILD_TAG="$CI_COMMIT_SHORT_SHA"

    - echo "Building $IMAGE:$BUILD_TAG"
    - |
      podman build \
        --label org.opencontainers.image.revision="$CI_COMMIT_SHA" \
        --label org.opencontainers.image.description="$(echo "$CI_COMMIT_MESSAGE" | head -n1)" \
        --label org.opencontainers.image.source="$CI_PROJECT_URL" \
        --label org.opencontainers.image.created="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        -t "$IMAGE:$BUILD_TAG" .

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
  * podman build
  * podman tag
  * podman push
8. Image available in registry


