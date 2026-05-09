---
title: "Request Certs from Smallstep CA"
date: 2026-05-9
tags:
  - pki
  - ci-cd
  - devops
---

## Goals
- remotely get Certs from StepCA Service
 - with step cli
 - with ACME certbot


## Using Step CLI

### Install Package

Install some basic utils adn the gpg key
```
sudo apt update
sudo apt install -y curl gnupg

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://packages.smallstep.com/keys/apt/repo-signing-key.gpg \
  | sudo tee /etc/apt/keyrings/smallstep.asc >/dev/null
```

Setup the Smallstep repo

```
sudo tee /etc/apt/sources.list.d/smallstep.sources >/dev/null <<'EOF'
Types: deb
URIs: https://packages.smallstep.com/stable/debian
Suites: debs
Components: main
Signed-By: /etc/apt/keyrings/smallstep.asc
EOF
```

Install the cli
```
sudo apt update
sudo apt install step-cli
```

### Setup the step CLI

Put RootCA in locla Trust Store
```
sudo wget -O /usr/local/share/ca-certificates/lab-root-ca.crt \
  https://ca.lab.lan/roots.pem \
  --no-check-certificate

sudo update-ca-certificates
```

Bootstrap step cli with URL and RootCA fingerprint
```
export FP=$(curl -s  https://ca.lab.lan/roots.pem | openssl x509 -noout -sha256 -fingerprint | awk -F'=' ' {gsub(":",""); print $2}')

step ca bootstrap \
  --ca-url https://ca.lab.lan \
  --fingerprint $FP
``