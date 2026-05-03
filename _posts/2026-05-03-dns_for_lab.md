---
layout: post
title: "Set up BIND for ProxMox lab"
date: 2026-05-03
author: "TimDude"
categories: ["Tech", "devops"]
tags: ["dns", "infrastruktur"]
---


So lets setup a DNS Server for our Lab

## Goals
- Proper name resolution of VMs inside the lab
- Bind TLS Cert to DNS Names instead IPs

## Install and configure Bind
Pretty lighrtweigth setup. Debian VM with 512M will do

### Install packages

We will install the daemon and some tooling
```
apt install bind9 bind9-utils dnsutils
```

### Configure Bind

We will enable forwarders so Lab VMs can not only resolve lab internal host
but also still use the internet for updates.
```
options {
    directory "/var/cache/bind";

    recursion yes;
    allow-recursion { 192.168.100.0/24; localhost; };
    listen-on { 127.0.0.1; 192.168.100.2; };
    listen-on-v6 { none; };

    forwarders {
        1.1.1.1;
        8.8.8.8;
    };

    dnssec-validation auto;
};
```
{: file="/etc/bind/named.conf.options" }

Define the Zones we want to have
```
zone "lab.lan" {
    type master;
    file "/etc/bind/db.lab.db
};
```
{: file="/etc/bind/named.conf.local

Finally create a Zone for the lab :)

```
$TTL 3600
@   IN SOA ns.lab.lan. admin.lab.lan. (
        2026050301
        3600
        900
        604800
        3600 )

@        IN NS ns.lab.lan.
ns       IN A  192.168.100.2
registry IN A  192.168.100.10
gitlab   IN A  192.168.100.11

```
{: file="/etc/bind/lab.lan.db










