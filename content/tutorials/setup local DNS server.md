---
title: "Setup Local DNS Server"
tags: 
- network
- China
- censorship
- GFW
---

## Overview
This tutorial is about how to get clean, optimized DNS resolve results by set up a local DNS server. Use China as an example.
## Tech Stack
- white-list mode: use public DNS server in China to resolve all domain we knew that have servers within China, all others use server outside China.
- dnsmasq
- DNSCrypt
- Debian latest stable (other Linux distributions works also)
## Setup
### [Debian Server Optimize](tutorials/Debian%20server%20optimize.md)

### dnsmasq
#### Install
``` bash
sudo apt update && sudo apt install dnsmasq
```
#### Change main configurations
```bash
sudo vi /etc/dnsmasq.conf
```
Update these configurations below:
1. Because the dnsmasq default configuration comes with all configurations commented out, you could just copy these below to the top of the configuration file;
2. Replace IP 10.20.30.40 below with IP of this machine that you want to use as the local DNS server.
```bash
port=53
no-resolv
no-poll
conf-dir=/etc/dnsmasq.d
#server=1.1.1.1
server=127.0.0.1#15353
listen-address=127.0.0.1,10.20.30.40

# performance improve
all-servers

cache-size=10000

# Do NOT cache failed search results.
no-negcache

# Do not set too high to reduce DNS cache problems.
min-cache-ttl=600

# Clear DNS cache when reloading /etc/resolv.conf.
clear-on-reload
```
#### Setup China domain whitelist mode
The configuration files is usually located in `/etc/dnsmasq.d/`, we use script to update it automatically here.

Create a folder to store related scripts:
```bash
sudo mkdir -p /opt/network/scripts
```
Create a script file:
```
cd /opt/network/scripts
sudo nano install_dnsmasq-china-list.sh
```
Add contents below:
1. "223.5.5.5 119.29.29.29 1.2.4.8" are some public DNS servers in China, please update according to your country or if you have more optimized one to use.
```bash
#!/bin/bash
set -e

WORKDIR="$(mktemp -d)"
SERVERS=(223.5.5.5 119.29.29.29 1.2.4.8)
# Not using best possible CDN pop: 1.2.4.8 210.2.4.8 223.5.5.5 223.6.6.6
# Dirty cache: 119.29.29.29 182.254.116.116

CONF_WITH_SERVERS=(accelerated-domains.china)
CONF_SIMPLE=(bogus-nxdomain.china)

echo "Downloading latest configurations..."
#git clone --depth=1 https://gitee.com/felixonmars/dnsmasq-china-list.git "$WORKDIR"
git clone --depth=1 https://github.com/felixonmars/dnsmasq-china-list.git "$WORKDIR"
#git clone --depth=1 https://pagure.io/dnsmasq-china-list.git "$WORKDIR"
#git clone --depth=1 https://github.com/felixonmars/dnsmasq-china-list.git "$WORKDIR"
#git clone --depth=1 https://bitbucket.org/felixonmars/dnsmasq-china-list.git "$WORKDIR"
#git clone --depth=1 https://gitlab.com/felixonmars/dnsmasq-china-list.git "$WORKDIR"
#git clone --depth=1 https://e.coding.net/felixonmars/dnsmasq-china-list.git "$WORKDIR"
#git clone --depth=1 https://codehub.devcloud.huaweicloud.com/dnsmasq-china-list00001/dnsmasq-china-list.git "$WORKDIR"
#git clone --depth=1 http://repo.or.cz/dnsmasq-china-list.git "$WORKDIR"

echo "Removing old configurations..."
for _conf in "${CONF_WITH_SERVERS[@]}" "${CONF_SIMPLE[@]}"; do
  rm -f /etc/dnsmasq.d/"$_conf"*.conf
done

echo "Installing new configurations..."
for _conf in "${CONF_SIMPLE[@]}"; do
  cp "$WORKDIR/$_conf.conf" "/etc/dnsmasq.d/$_conf.conf"
done

for _server in "${SERVERS[@]}"; do
  for _conf in "${CONF_WITH_SERVERS[@]}"; do
    cp "$WORKDIR/$_conf.conf" "/etc/dnsmasq.d/$_conf.$_server.conf"
  done

  sed -i "s|^\(server.*\)/[^/]*$|\1/$_server|" /etc/dnsmasq.d/*."$_server".conf
done

echo "Restarting dnsmasq service..."
if hash systemctl 2>/dev/null; then
  systemctl restart dnsmasq
elif hash service 2>/dev/null; then
  service dnsmasq restart
elif hash rc-service 2>/dev/null; then
  rc-service dnsmasq restart
elif hash busybox 2>/dev/null && [[ -d "/etc/init.d" ]]; then
  /etc/init.d/dnsmasq restart
else
  echo "Now please restart dnsmasq since I don't know how to do it."
fi

echo "Cleaning up..."
rm -r "$WORKDIR"
```

Setup permission of the script and run once:
```bash
script_file="/opt/network/scripts/install_dnsmasq-china-list.sh"
sudo chmod +x "$script_file"
"$script_file"
```

Update the China domain list every week:
```bash
echo 3 3 \* \* 6   root   \"/opt/network/scripts/install_dnsmasq-china-list.sh\" > /etc/cron.d/dnsmasq
```

If you want the server itself to use this DNS also, could run this:
```bash
echo nameserver 127.0.0.1 > /etc/resolv.conf
```
Notes: some OS might change the resolve.conf automatically and needs a corresponding way to update DNS setting instead of edit directly.

Reference:
- https://github.com/felixonmars/dnsmasq-china-list

### dnscrypt-proxy
#### Install
```bash
sudo apt update && sudo apt install dnscrypt-proxy
```
#### Setup profile
```bash
cd /etc/dnscrypt-proxy
sudo cp dnscrypt-proxy.toml dnscrypt-proxy.toml.original
sudo nano dnscrypt-proxy.toml
```
Add contents below:
```bash
listen_addresses = ['127.0.0.1:15353']
server_names = ['cloudflare']
cache = false

[query_log]
  file = '/var/log/dnscrypt-proxy/query.log'

[nx_log]
  file = '/var/log/dnscrypt-proxy/nx.log'

[sources]
  [sources.'public-resolvers']
  url = 'https://download.dnscrypt.info/resolvers-list/v2/public-resolvers.md'
  cache_file = '/var/cache/dnscrypt-proxy/public-resolvers.md'
  minisign_key = 'RWQf6LRCGA9i53mlYecO4IzT51TGPpvWucNSCh1CBM0QTaLn73Y7GFO3'
  refresh_delay = 72
  prefix = ''
```
Update startup files and restart:
```bash
sudo systemctl mask dnscrypt-proxy.socket
sudo systemctl disable dnscrypt-proxy-resolvconf.service

# comment out all lines with ".socket"
sudo systemctl edit --full dnscrypt-proxy.service

sudo systemctl daemon-reload
sudo systemctl restart dnscrypt-proxy
```
References:
- https://wzyboy.im/post/1372.html
- https://github.com/dnscrypt/dnscrypt-proxy/wiki/Installation-on-Debian-and-Ubuntu
### Check
Use dig to check DNS resolve status.

Install packages:
```bash
sudo apt update && sudo apt install dnsutils
```
Check different DNS ports:
```bash
dig google.com @127.0.0.1 -p 53
dig google.com @127.0.0.1 -p 15353
```