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
Pretty lighrtweigth setup. Debian Netinstall VM with 256M RAM will do

### Install packages

We will install the daemon and some tooling
```
apt install bind9 bind9-utils dnsutils
```

### Configure Bind

We will enable forwarders so Lab VMs can not only resolve lab internal host
but also still use the internet for updates.

Our lab lan is 192.168.100.0/24, the DNS VM ip is 192.168.100.2 and bind will listen on that adress as well. Forwarders will be Cloudflare and google.

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
    file "/etc/bind/lab.lan.db
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
{: file="/etc/bind/lab.lan.db"}

### Start Service 
Check and restart sercvice

```
named-checkconf
named-checkzone lab.lan /etc/bind/db.lab.lan
systemctl restart bind9
systemctl status bind9
```

## Test from a Lab vm

Now we got the DNS Server up an running. 
Lets test from a Clinet if it can resolve local and internet hosts

### Setup resolver

On Debian i did this
```
search lab.lan
nameserver 192.168.100.2

```
{: file="/etc/resolv.conf"}


### Test Name resolution
One shot at the interwebs one local

```
root@registry:# nslookup google.de
Server:		192.168.100.2
Address:	192.168.100.2#53

Non-authoritative answer:
Name:	google.de
Address: 142.251.20.94

root@registry:# nslookup gitlab
Server:		192.168.100.2
Address:	192.168.100.2#53

Name:	gitlab.lab.lan
Address: 192.168.100.11
```






