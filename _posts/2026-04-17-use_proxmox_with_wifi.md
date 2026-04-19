---
layout: post
title: "Using ProxMox with WiFi"
date: 2026-04-17
author: "TimDude"
categories: ["Tech","Hypervisors"]
tags: ["proxmox", "linux", "networking"]
---

If you're on the go with your LTE HotSpot, can't run cable in your home or have any other reason to use WiFi with ProxMox.
I'll shed some light on your options.

WiFi interfaces cannot be bridged like Ethernet in most setups, so when using onboard WiFi you must use routing or DNAT instead of bridging.

# Options
* The simplest, a WiFi bridge like TPLINK WR802N you can plug into your ProxMox ethernet Port (bridging can work)
* Install wpasupplicant and use the WiFI on your ProxMox box (bridging wont work)
* Use USB Tethering of you LTE HotSpot or Android Phone (bridging wont work)

The first is pretty simple an straight forward, so i will look at the second option of using onboard WiFi.

ProxMox setup wont configure WiFi you will have to do that your self and need to install some packages in the process.
To access packages you will need an internet connection. 

## Temp inet access
You can temporarily connect with ethernet to a router or use USB tethering.
Either of them can be activated by the installer. But also manually.

```
ip link set enx00e04c010203 up
dhclient enx00e04c010203
```

## Get packages
```
apt install wpasupplicant iptables-persistent
```

## Configure wpa-suplicant
```
ctrl_interface=/run/wpa_supplicant
update_config=1
country=DE

network={
    ssid="YOUR_WIFI_NAME"
    psk="YOUR_PASSWORD"
}
```
{: file="/etc/wpa_supplicant/wpa_supplicant.conf" }

Optional generate hashed psk
```
wpa_passphrase YOUR_WIFI_NAME YOUR_PASSWORD
```

## Make the interface come up on boot
Check with ip link for the name of your WiFi interface.
It could be wlan0, wlp2s0, or something similar.
```
auto wlp2s0
iface wlp2s0 inet dhcp
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```
{: file="/etc/network/interfaces" }

Only your WiFI should have default route.
Remove default gateway and brdiging from vmbr0
```
auto vmbr0
iface vmbr0 inet static
    address 10.10.10.1/24
    bridge_ports none
    bridge_stp off
    bridge_fd 0
```
{: file="/etc/network/interfaces" }

## The DNAT approach
### Make VMs able to use the internet
On proxmox pve
IP Forwarding
```
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-ip_forwarding.conf
sysctl --system
```
{: file="/etc/sysctl.d/99-ip_forwarding.conf" }

Iptables NAT and Forwarding Rules
```
export wifi="wlp2s0"

iptables -t nat -A POSTROUTING -o ${wifi} -j MASQUERADE
iptables -A FORWARD -i vmbr0 -o ${wifi} -j ACCEPT
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT

iptables-save > /etc/iptables/rules.v4
```

### Access VMs via Port Forwarding
To access them (e.g. via SSH / HTTPS), you can forward ports on the Proxmox host.

Example
```
Proxmox (WiFi): 192.168.0.105
VM: 192.168.100.10
Goal SSH: 192.168.0.105:1022 → 192.168.100.10:22
Goal HTTPS: 192.168.0.105:10443 → 192.168.100.10:443
```

On proxmox pve
NAT and forwarding rules
```
export wifi="wlp2s0"

iptables -t nat -A PREROUTING -i ${wifi} -p tcp --dport 1022 -j DNAT --to-destination 192.168.100.10:22
iptables -A FORWARD -i ${wifi} -o vmbr0 -p tcp -d 192.168.100.10 --dport 22 -j ACCEPT

iptables -t nat -A PREROUTING -i ${wifi} -p tcp --dport 10443 -j DNAT --to-destination 192.168.100.10:443
iptables -A FORWARD -i ${wifi} -o vmbr0 -p tcp -d 192.168.100.10 --dport 443 -j ACCEPT

iptables-save > /etc/iptables/rules.v4
```

Try to connect
```
ssh -p 1022 user@192.168.0.105
```

## The routing Approach
This one is more flexible, than NAT.
In case you use a mobile router you will often need to ad a static route to your laptop.
On most portable routers you cant configure static routes.

### Make VMs able to use the internet
On proxmox pve

IP Forwarding
```
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-ip_forwarding.conf
sysctl --system
```
{: file="/etc/sysctl.d/99-ip_forwarding.conf" }

Iptables NAT and Forwarding
```
export wifi="wlp2s0"

iptables -t nat -A POSTROUTING -o ${wifi} -j MASQUERADE
iptables -A FORWARD -i vmbr0 -o ${wifi} -j ACCEPT
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT

iptables-save > /etc/iptables/rules.v4
```

### Add inbound rules to acces VMs

```
export wifi="wlp2s0"

iptables -A FORWARD -i ${wifi} -o vmbr0 -p tcp -d 192.168.100.10 --dport 22 -j ACCEPT
iptables -A FORWARD -i ${wifi} -o vmbr0 -p tcp -d 192.168.100.11 --dport 22 -j ACCEPT

iptables-save > /etc/iptables/rules.v4
```

### Add static route to Laptop

Via will be the IP Proxmox has on your WiFi
```
sudo ip route add 192.168.100.0/24 via 192.168.0.105
```







