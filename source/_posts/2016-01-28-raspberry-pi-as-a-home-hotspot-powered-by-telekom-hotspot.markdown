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
```
sudo iptables -t nat -A POSTROUTING -o wlan1 -j MASQUERADE
```
```
sudo iptables -A FORWARD -i wlan1 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```
```
sudo iptables -A FORWARD -i wlan0 -o wlan1 -j ACCEPT 
```

6. Save the rules: 
```
sh -c iptables-save > /etc/iptables.ipv4.nat
```
7. Run dhcp server and hostapd: 
```
sudo update-rc.d hostapd enable
```
```
sudo update-rc.d isc-dhcp-server enable 
```

8. You should be able now to connect to raspberry PI wifi network(wlan0) from home. But no internet yet.
8. Now connect to telekom hotspot: 
```
sudo iwconfig wlan1 essid Telekom; sudo dhcpclient wlan1
```
10. Move to your laptop and perform http Login with your Telekom username and password. In Chrome open console and in a network tab find the post to Telekom gateway.
Click on that request and click "save as curl command" paste the command to a script and save on raspberry.
Then write short script that will check internet connection and if there is no it will reauthenticate. Sth like this
```
#!/bin/sh
while [ 1 ]; do
    sudo ping www.google.com -c 1 && echo "OK" || curl -v 'https://hotspot.t-mobile.net/wlan/rest/login' -H 'Origin: https://hotspot.t-mobile.net' -H 'Accept-Encoding: gzip, deflate' -H 'Accept-Language: en-US,en;q=0.8' -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/46.0.2490.86 Safari/537.36' -H 'Content-Type: application/json;charset=UTF-8' -H 'Accept: application/json, text/plain, */*' -H 'Referer: https://hotspot.t-mobile.net/TD/hotspot/de_DE/index.html?origurl=http%3A%2F%2Fwww.kimeta.de%2FOfferDetail.aspx&ts=1451396580457' -H 'Cookie: 06847EADCEBFD2E920C1F12378924815.P2; oam.Flash.RENDERMAP.TOKEN=118ogwrec4' -H 'Connection: keep-alive' --data-binary '{"username":"phone-number@t-mobile.de","password":"password","rememberMe":true}' --compressed
    sleep 1
    done
```
Put the script in monit or run with nohup in raspberry pi
{% img /images/raspberry.jpeg %}
