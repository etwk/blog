---
title: "Debian Server Optimize"
tags: 
- network
- Linux
- Debian
- OS
---

## Overview
Debian OS as a server can be optimized in a lot of places, to list some important ones:
- Change TCP congestion control to use BBR or others, to improve network connectivity and bandwidth
- Enlarge file limit

This applies to other Linux distribution also. 

Optimization listed here are just for reference to give reader an idea how things could be done:
- They are not a guide book that you should follow absolutely;
- Different use cases and different OS might need some changes.
## Setup
### Basic software
Install some helpful tools:
```bash
sudo apt update
sudo apt install htop vnstat
```
### Security
#### ufw
ufw is a firewall on Linux which can provide some basic and necessary network protection. 

CATION: careful with the setup, otherwise you could block your own network access to the server.

Install:
```
sudo apt update && sudo apt install ufw
```
Add some basic whitelists:
```bash
# Use this if have some trusted IP addresses and want unrestricted access from these IP addresses.
sudo ufw allow from a.b.c.d
# Allow access to SSH so that you can access to this server from any place via SSH.
# CATION: this rule is less secure because it opens SSH to the Internet, you might want to restrict the access more.
sudo ufw allow ssh
```
#### fail2ban
Install:
```bash
sudo apt update && sudo apt install fail2ban
```
### System & Network
Change congestion control to BBR, and optimize system for better network performance. 

BBR is a advanced congestion control built into Linux kernel (needs high version):
- It can help improve connectivity of TCP services a lot, sometimes you could see 100x improve. 
- There are similar products you could try, like lotServer.

Change settings:
```
sudo vi /etc/sysctl.d/local.conf
```
Add these settings if they are not exists elsewhere:
```bash
fs.file-max = 65534

net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
net.core.rmem_default = 1342177
net.core.wmem_default = 1342177
net.core.netdev_max_backlog = 300000
net.core.somaxconn = 4096

net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.ip_local_port_range = 10000 65000
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_rmem = 4096 1342177 67108864
net.ipv4.tcp_wmem = 4096 1342177 67108864
net.ipv4.tcp_mtu_probing = 1

net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```
Enable the settings above:
```bash
sudo sysctl --system
```
Check if BBR enabled:
```bash
lsmod | grep bbr

# You should see something like this below if BBR are enabled.
# tcp_bbr                20480  35
```
#### Change Open File Limit
Open file:
```bash
sudo vi /etc/security/limits.conf
```
Add these settings to top of the file:
```bash
root soft nofile 65534
root hard nofile 65534

* soft nofile 65534
* hard nofile 65534
```
#### Reboot & Check
Reboot the server:
```bash
sudo reboot
```
After reboot, check if you can log in to the server and everything works correctly.