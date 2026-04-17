---
layout: post
title: "Using ProxMox with WiFi"
date: 2026-04-17
author: "TimDude"
categories: ["Tech"]
tags: ["proxmox", "linux", "networking"]
---

If youre on the go with your LTE HotSpot, cant run cable in your home or have any other reason to use WiFi with ProxMox, I'll shed some light on your options.

Depending with which option you go, you will have working bridging or will have to use NAT or routing to connect to VMs.

# Options
* The simplest, a WiFi bridge like TPLINK WR802N you can plug into your ProxMox ethernet Port (bridging can work)
* Install wpasupplicant and user the WiFI on your ProxMox box (bridging wont work)
* Use USB Tethering of you LTE HotSpot or Android Phone (bridging wont work)

The first ist pretty simple an straight forward, so i will look at the second option of using onbard WiFi.
I tethered my Android phone temporarily to install packages from the internet, so it will also be described in the process.

ProxMox setup wont configure WiFi you will have to do that your self and need to install some packages in the process.
To access packages you will need an internet connection. 

## Temp inet access
You can temporaily connect with ethernet to a router or use USB tethering.
Either of them can be activated by the installer. But also manually.

```
ip link set enx00e04c010203 up
dhclient enx00e04c010203
```

## Get packages
```
apt get wpasupplicant iptable-persistent
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

## Make interface come up on boot
The builtin WiFi iface could be wlan0 / wlp2s0 or similar
```
auto wlan0
iface wlan0 inet dhcp
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```
{: file="/etc/network/interfaces" }

## Make VMs able to use the internet

IP Forwarding
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf.d/99-ip_forwarding.conf
sysctl --system
```

NAT Rule
```
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
iptables-save > /etc/iptables/rules.v4
```







