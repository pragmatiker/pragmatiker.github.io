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

### Set UP the Root Certificate Authority

First the Root CA.
Store this offline, give it a strong Password.

```
sudo mkdir -p /opt/step-root
cd /opt/step-root

step certificate create "My Root CA" root.crt root.key \
--profile root-ca --kty RSA --size 4096
```

### Set up the Intermediate Certificate Authority

Sign the Intermediate with the existing Root CA and generate a Config.
You will be asked for the Root Ca Passphrase and for a Passphrase for the new,
intermediate CA.

```
mkdir -p /opt/step-intermediate
cd /opt/step-intermediate

export STEPPATH=/opt/step/ca

step ca init \
  --name "My Existing CA" \
  --root /opt/step-root/root.crt \
  --key /opt/step-root/root.key \
  --dns ca.lab.lan \
  --address :443 \
  --deployment-type standalone \
  --provisioner ca@lab.lan
```

### Alter the Config to suite Container 

Store the Intermediate passphrase
```
echo 'CaPa$$word' > $STEPPATH/secrets/password
```

Rewrite host paths to container paths 
```
sed -i "s|$STEPPATH|/home/step|g" $STEPPATH/config/ca.json
sed -i "s|$STEPPATH|/home/step|g" $STEPPATH/config/defaults.json
```

### Set up the podman container

Install podman
```
apt install -y podman
```

Add a system user for stepca and give it acces to files.
```
sudo useradd --system --uid 2000 --home $STEPPATH --shell /usr/sbin/nologin stepca
sudo chown -R 2000:2000 $STEPPATH
```

```
podman rm -f step-ca
podman run -d \
  --name step-ca \
  -p 443:443 \
  -v $STEPPATH:/home/step:Z \
  --user 2000:2000 \
  --restart=unless-stopped \
  --health-cmd="step-ca health https://127.0.0.1:443" \
  docker.io/smallstep/step-ca \
  step-ca /home/step/config/ca.json --password-file /home/step/secrets/password
```

## Setup trust on clients


For Debian / Ubuntu
```
sudo wget -O /usr/local/share/ca-certificates/lab-root-ca.crt \
  https://ca.lab.lan/roots.pem \
  --no-check-certificate

sudo update-ca-certificates
```