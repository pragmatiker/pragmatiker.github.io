---
title: "Set up a small Certificate Authority with stepca"
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

Set up the repo

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
export ROOTCA=/opt/RootCA
sudo mkdir -p $ROOTCA
cd $ROOTCA

step certificate create "My Root CA" root.crt root.key \
--profile root-ca --kty RSA --size 4096
```

### Set up the Intermediate Certificate Authority

This will contain private keys and password files.
```
export STEPPATH=/opt/step-ca
mkdir -p $STEPPATH/secrets
```

Store the Intermediate CA key password.
```
umask 0077; echo 'CaPa$$word' > $STEPPATH/secrets/ca_key_password
```

The provisioner password is only needed during initialization.
The running CA only requires access to the Intermediate CA key password.
Put this in PW Manager. We will delete this file after init.
```
echo 'ProvPa$$word' > $STEPPATH/secrets/provisioner_password
```

Sign the Intermediate with the existing Root CA and generate a Config.
You will be asked for the Root Ca Passphrase.
Provisioner and Ca key will be passed in via the files generated dearlier

```
step ca init \
  --name "My Existing CA" \
  --root $ROOTCA/root.crt \
  --key $ROOTCA/root.key \
  --dns ca.lab.lan \
  --address :443 \
  --deployment-type standalone \
  --provisioner ca@lab.lan \
  --password-file $STEPPATH/secrets/ca_key_password \
  --provisioner-password-file $STEPPATH/secrets/provisioner_password
rm -v $STEPPATH/secrets/provisioner_password
```

### Alter the Config to suit the Container 

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
sudo chmod 600 $STEPPATH/secrets/*
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
  step-ca /home/step/config/ca.json --password-file /home/step/secrets/ca_key_password
```

## Setup trust on clients

### Install the RootCA in the local trust store

For Debian / Ubuntu
```
sudo wget -O /usr/local/share/ca-certificates/lab-root-ca.crt \
  https://ca.lab.lan/roots.pem \
  --no-check-certificate

sudo update-ca-certificates
```

### Test SSL trust with curl

```
curl https://ca.lab.lan/health
```

expected result
```
{"status":"ok"}
```
