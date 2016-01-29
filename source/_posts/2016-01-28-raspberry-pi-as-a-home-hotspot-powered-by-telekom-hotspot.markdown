---
layout: post
title: "Raspberry PI as a home hotspot powered by Telekom Hotspot"
date: 2016-01-28 06:50:15 +0000
comments: true
categories: 
---

If you are living in Germany and you have phone plan covering free Telekom Hotspot usage probably you wondered, like me if its possible to broadcast signal from hotspot to home.
One function that prohibits us from using simple reapeter is the fact that Telekom hotspot has html username/password authentication. The hotspot deuthenticates the user when inactive and when token expires.
The idea is to put 2 WIFI cards into raspberry PI. One will receive signal from hotspot. On another we will configure hostapd and dhcp server to broadacast signal at home. We will write simple script that authenticates user in Telekom hotspot using curl.

1. Step is to get raspberry PI. I'm using the B+ model that has 4 USB exists and microSD slot.
2. Install Raspbian OS
3. Configure raspberry PI
```
sudo apt-get update
sudo apt-get -y install hostapd isc-dhcp-server iptables wpa_supplicant
```
edit the following files: ```/etc/hostapd/hostapd.conf```,  ```/etc/dhcp/dhcpd.conf```, ```/etc/default/isc-dhcp-server```, ```/etc/network/interfaces```:
4. Enable packet forwarding on start by adding the line "net.ipv4.ip_forward=1" to  " /etc/sysctl.conf"
5. Configure IP-Tables(I assume wlan1 is a interface connected to Telekom Hotspot.
sudo iptables -t nat -A POSTROUTING -o wlan1 -j MASQUERADE
sudo iptables -A FORWARD -i wlan1 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o wlan1 -j ACCEPT 

6. Save the rules: sh -c "iptables-save > /etc/iptables.ipv4.nat"
7. Run dhcp server and hostapd: 
sudo update-rc.d hostapd enable
sudo update-rc.d isc-dhcp-server enable 

8. You should be able now to connect to raspberry PI wifi network(wlan0) from home. But no internet yet.
8. Now connect to telekom hotspot: sudo iwconfig wlan1 essid Telekom; sudo dhcpclient wlan1
10.

