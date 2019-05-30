---
layout: post-header-image
title: "Wireless hardware VPN gateway"
description: "Build a portable, Raspberry Pi-based VPN router to encrypt your traffic on any untrusted public or private network."
category: "play"
image:
  head: vpn_tight.png
tags: [DIY]
---

### What's needed

* 1 x Raspberry Pi 3, with SD-card, power supply and optional case
* 1 x external WiFi dongle, preferrably with a coaxial antenna

We need 2 WiFi adapters, one to connect to the Internet, and one to create the network that our devices are going to connect to. The Pi will run a VPN client, a DHCP host, a network broadcaster and some captive portal software; it will act as a secure gateway between any device and the insecure Internet connection provided by the access point. The first adapter can be generic, but the second (the one that broadcasts the network) must be master mode capable. This is the case of the internal WiFi adapter in the Raspberry Pi 3. It doesn't have great reception, having a very tiny antenna, however we assume that we are more likely to have the VPN gateway close to the client devices, and far from the router that provides internet access. In that scenario it makes sense to use the internal adapter for broadcast, and the high-power adapter to connect to Internet. In all the commands below, `wlan0` is the internal adapter, `wlan1` is the WiFi dongle.

### Preparing the Pi

We start by installing the necessary packages:

```bash
sudo apt-get update
sudo apt-get install hostapd dnsmasq openvpn lighttpd php5-cgi conntrack
```

* `hostapd` will allow us to act as access point and share our connection
* `dnsmasq` will manage internal IP addresses and DNS
* `openvpn` is our VPN client
* `lighttpd` is a very light server to display our captive portal welcome page
* `conntrack` tracks repeated connections by clients, it will implement our captive portal

#### Configuring the access point

First, we must configure our `/etc/network/interfaces` to assign our two interfaces their respective roles. Here `wlan0` is the internal WiFi adapter that will accept connections from client machines, `wlan1` is the external USB-powered adapter.

```
source-directory /etc/network/interfaces.d

auto lo
iface lo inet loopback

iface eth0 inet manual

allow-hotplug wlan0
iface wlan0 inet static
address 192.168.10.1
netmask 255.255.255.0
network 192.168.10.0

allow-hotplug wlan1
iface wlan1 inet manual
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

```

Then, configure `/etc/wpa_supplicant/wpa_supplicant.conf` as usual to allow your Pi to connect to your local network.

We now edit the main config file for `hostapd`, located at `/etc/hostapd/hostapd.conf`:

```
interface=wlan0
driver=nl80211
ssid=mySSID
hw_mode=g
channel=7
wpa_passphrase=myPassword

ieee80211n=1
wmm_enabled=1
ht_capab=[HT40][SHORT-GI-20][DSSS_CCK-40]
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```

The main fields to modify are `ssid` and `wpa_passphrase`, which describe the network that `wlan0` is going to broadcast. You may also need to change the `driver` field if you did not use the internal `wlan0` adapter or if your Pi has a different hardware than mine.

Now add the following line:

```
denyinterfaces wlan0
```

to your `/etc/dhcpcd.conf`. This will prevent `dhcpcd` from seeking automatic DHCP on that adapter (that's client behaviour, reserved for `wlan1`).

Now we configure the access point itself. We need to tell `dnsmasq`, our DNS and DHCP server, the address we would like to give the gateway, the DNS server it should use (we use `8.8.8.8`, Google), and the range of addresses it can attribute to its clients. This is all specified in your `/etc/dnsmasq.conf` :
```
interface=wlan0
listen-address=192.168.10.1
bind-interfaces
server=8.8.8.8
domain-needed
bogus-priv
dhcp-range=192.168.10.100,192.168.10.200,12h
```

Then we start everything:

```
sudo dhcpcd restart
sudo service hostapd restart
sudo service dnsmasq start
sudo update-rc.d dnsmasq enable
```

### Configuring the VPN

This is a bit tricky, as we want the VPN to act as a "man in the middle" between our two `wlan` interfaces.