---
title: "Set up Gitlab"
date: 2026-05-7
tags:
  - pki
  - ci-cd
  - devops
---


So lets setup a small CA so we can use SSL in our lab


## Goals
- Two Tier setup Root CA & Intermediate CA
- Put Root CA in lab TrustStores
- Test with curl

## Setup the CA
### Install CLI Tools

Install the gpg key

```
sudo apt update
sudo apt install -y curl gnupg

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://packages.smallstep.com/keys/apt/repo-signing-key.gpg \
  | sudo tee /etc/apt/keyrings/smallstep.asc >/dev/null
```

Setu the repo

```
sudo tee /etc/apt/sources.list.d/smallstep.sources >/dev/null <<'EOF'
Types: deb
URIs: https://packages.smallstep.com/stable/debian
Suites: debs
Components: main
Signed-By: /etc/apt/keyrings/smallstep.asc
EOF
```

Install the helper tools.
The CA we will actually run in a Container, not as Daemon on the host.
```
sudo apt update
sudo apt install step-cli
```

### Set UP the Root Certificate Authorotie

First the Root CA

```
sudo mkdir -p /opt/step-root
cd /opt/step-root

step certificate create "My Root CA" root.crt root.key \
--profile root-ca --kty RSA --size 4096
```

### Set up the Intermediate Certificate Authorotie

Sign the Intermediate with the Root CA

```
mkdir -p /opt/step-intermediate
cd /opt/step-intermediate

step certificate create "My Intermediate CA" intermediate.crt intermediate.key \
--profile intermediate-ca --kty RSA --size 4096 \
--ca /opt/step-root/root.crt --ca-key /opt/step-root/root.key
```



### Set up the podman container

```
apt install -y podman
```

```
podman run -d \
  --name step-ca \
  -p 9000:9000 \
  -v /opt/step/ca:/home/step:Z \
  --restart=unless-stopped \
  --health-cmd="step-ca health https://127.0.0.1:9000" \
  docker.io/smallstep/step-ca \
  step-ca /home/step/config/ca.json
```
