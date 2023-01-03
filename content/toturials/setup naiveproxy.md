---
title: "Setup naiveproxy"
tags: 
- network
- China
- censorship
- GFW
---

## Overview
This tutorial is about how to set up naiveproxy server and client to provide transparent proxy to a local network.
## Tech Stack
- white-list mode: all TCP access to IP addresses outside China goes through naiveproxy, others go direct.
- NaïveProxy
- iptables
- Debian latest stable (other Linux distributions works also)
## Prepare
- A VPS server in foreign country:
	- with unrestricted Internet access
	- has good network connectivity to your network in China: low package loss, low latency, high bandwidth and stable
- A domain name (if you don't already have one, could register one in NameSilo, they have reasonable price and contains privacy protection by default)
- Domain DNS hosting (DigitalOcean and many other providers offer this for free)
## Setup
Please also check official documents for updated information: https://github.com/klzgrad/naiveproxy
### [Debian Server Optimize](../toturials/Debian%20server%20optimize.md)
#### System settings for naiveproxy
```
sudo vi /etc/sysctl.d/local.conf
```

Add or update settings below:
```bash
net.ipv4.tcp_slow_start_after_idle=0
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_notsent_lowat=16384
net.ipv4.ip_forward = 1
```

### Server
#### [Obtain SSL Certificate](.//SSL%20certificate.md)
#### Install go
https://golang.org/doc/install
#### Install caddy with Naive
```bash
# go to a folder where you want to store the related files
mkdir ~/folder/naive && cd "$_"

go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
~/go/bin/xcaddy build --with github.com/caddyserver/forwardproxy@caddy2=github.com/klzgrad/forwardproxy@naive
sudo setcap cap_net_bind_service=+ep ./caddy
mv caddy /usr/local/bin/
```

Prepare web files the caddy server going to serve:
```bash
sudo mkdir -p /var/www/html
sudo touch /var/www/html/index.html
```

Prepare caddy configuration file:
```bash
sudo mkdir /etc/caddy
sudo vi /etc/caddy/caddy.conf
```

Update with below:
1. Change settings based on features you need.
2. IMPORTANT! Replace:
	1. Domain address to your own;
	2. The 2 path to certificate files;
	3. basic_auth.
```bash
{
  admin off
  log {
      output file /var/log/caddy/access.log
      level INFO
  }
  servers :443 {
      protocol {
          experimental_http3
      }
  }
}

:80 {
  redir https://{host}{uri} permanent
}

:443, abc.exampleDomain.com
tls "/etc/ssl/cert.d/abc.exampleDomain.com/fullchain.pem" "/etc/ssl/cert.d/abc.exampleDomain.com/privkey.pem"
route {
  forward_proxy {
      basic_auth SPo5lLwplaLe2Nco Aopx3ZN8KqA6kWEtAjbSwiHbxhP1ZyYr
      hide_ip
      hide_via
      probe_resistance
  }
  file_server {
      root /var/www/html
  }
}
```

Update caddy start file:
```bash
sudo vi /etc/systemd/system/caddy.service
```
```bash
[Unit]
Description=Caddy HTTP/2 web server
After=network-online.target
Wants=network-online.target systemd-networkd-wait-online.service
StartLimitIntervalSec=14400
StartLimitBurst=10

[Service]
User=root
Group=root
ExecStart=/usr/local/bin/caddy run --config /etc/caddy/caddy.conf --adapter caddyfile
ExecReload=/usr/bin/kill -USR1 $MAINPID

# Do not allow the process to be restarted in a tight loop. If the
# process fails to start, something critical needs to be fixed.
Restart=on-abnormal

# Use graceful shutdown with a reasonable timeout
KillMode=mixed
KillSignal=SIGQUIT
TimeoutStopSec=5s

LimitNOFILE=1048576
LimitNPROC=512

[Install]
WantedBy=multi-user.target
```

Start caddy server and enable auto start:
```bash
sudo systemctl start caddy && sudo systemctl enable caddy
```

### Client
#### Overview
- Use HAProxy to load balancing between multiple servers if you need more than one server.
- Use iptables for load balancing between multiple naiveproxy instances.
	- At current writing, one naiveproxy instance allows up to around 250 connections, we could add concurrency up to 4, that allows 250 x 4 connections.
	- If that's still not enough, we can set up multiple naiveproxy instances. (We are going for 3 instances in this tutorial.)
	- Lower concurrency and instances of clients to the number you need.
	- IMPORTANT: setup multiple connections or instances might be less secure because it lowers anonymity.
#### Network setup
Create a directory to store configuration files:
```bash
sudo mkdir /etc/network.d
```

Add IP addresses of all upstream naiveproxy servers to ignore file:
- Replace IP addresses to your own.
```bash
sudo echo a.a.a.a >> /etc/network.d/server_ignore
sudo echo b.b.b.b >> /etc/network.d/server_ignore
```

Get IP range list of all within China:
```bash
sudo wget -O "/etc/network.d/CN_IPv4_blocks" https://raw.githubusercontent.com/17mon/china_ip_list/master/china_ip_list.txt
```
todo: update the list automatically.

#### Naiveproxy client
Install packages:
```bash
sudo apt update && sudo apt install libnss3
```
Prepare execution file:
```bash
# Go to a directory to prepare files
cd ~/folder

# Get link to the latest version for your plantform from https://github.com/klzgrad/naiveproxy/releases
wget https://github.com/klzgrad/naiveproxy/releases/download/v108.0.5359.94-1/naiveproxy-v108.0.5359.94-1-linux-x64.tar.xz

tar -xvf naiveproxy*tar.xz
sudo cp naive /usr/local/bin/naiveproxy
```
Prepare config files:
- IMPORTANT: change domain name and auth info to your corresponding ones.
```bash
sudo mkdir -p /etc/naiveproxy/conf.d

sudo vi /etc/naiveproxy/conf.d/1.json
```
```bash
{
  "listen": "redir://0.0.0.0:2081",
  "host-resolver-rules": "MAP abc.exampleDomain.com:443 127.0.0.1:1443",
  "proxy": "https://SPo5lLwplaLe2Nco:Aopx3ZN8KqA6kWEtAjbSwiHbxhP1ZyYr@abc.exampleDomain.com",
  "insecure-concurrency": "4",
    "log": "/var/log/naiveproxy.log"
}
```
```bash
sudo vi /etc/naiveproxy/conf.d/2.json
```
```bash
{
  "listen": "redir://0.0.0.0:2082",
  "host-resolver-rules": "MAP abc.exampleDomain.com:443 127.0.0.1:1443",
  "proxy": "https://SPo5lLwplaLe2Nco:Aopx3ZN8KqA6kWEtAjbSwiHbxhP1ZyYr@abc.exampleDomain.com",
  "insecure-concurrency": "4",
    "log": "/var/log/naiveproxy.log"
}
```
```bash
sudo vi /etc/naiveproxy/conf.d/3.json
```
```bash
{
  "listen": "redir://0.0.0.0:2083",
  "host-resolver-rules": "MAP abc.exampleDomain.com:443 127.0.0.1:1443",
  "proxy": "https://SPo5lLwplaLe2Nco:Aopx3ZN8KqA6kWEtAjbSwiHbxhP1ZyYr@abc.exampleDomain.com",
  "insecure-concurrency": "4",
    "log": "/var/log/naiveproxy.log"
}
```
todo: log rotation.

Prepare network startup file:
```bash
sudo vi /etc/naiveproxy/network.sh
```
```bash
#!/bin/sh

#create a new chain named NAIVEPROXY
iptables -t nat -N NAIVEPROXY

# Ignore your naiveproxy server's addresses
# It's very IMPORTANT, be careful.

for server_ip in `cat /etc/network.d/server_ignore`;
do
    iptables -t nat -A NAIVEPROXY -d "${server_ip}" -j RETURN
done

# Ignore LANs IP address
iptables -t nat -A NAIVEPROXY -d 0.0.0.0/8 -j RETURN
iptables -t nat -A NAIVEPROXY -d 10.0.0.0/8 -j RETURN
iptables -t nat -A NAIVEPROXY -d 127.0.0.0/8 -j RETURN
iptables -t nat -A NAIVEPROXY -d 169.254.0.0/16 -j RETURN
iptables -t nat -A NAIVEPROXY -d 172.16.0.0/12 -j RETURN
iptables -t nat -A NAIVEPROXY -d 192.168.0.0/16 -j RETURN
iptables -t nat -A NAIVEPROXY -d 224.0.0.0/4 -j RETURN
iptables -t nat -A NAIVEPROXY -d 240.0.0.0/4 -j RETURN

# Ignore China IP address
for white_ip in `cat /etc/network.d/CN_IPv4_blocks`;
do
    iptables -t nat -A NAIVEPROXY -d "${white_ip}" -j RETURN
done

# Anything else should be redirected to naiveproxy's local port
iptables -t nat -A NAIVEPROXY -p tcp -j REDIRECT --to-ports 2081  -m statistic --mode nth --every 3 --packet 0
iptables -t nat -A NAIVEPROXY -p tcp -j REDIRECT --to-ports 2082  -m statistic --mode nth --every 2 --packet 0
iptables -t nat -A NAIVEPROXY -p tcp -j REDIRECT --to-ports 2083 

# Apply the rules
iptables -t nat -A PREROUTING -p tcp -j NAIVEPROXY
iptables -t nat -A OUTPUT -p tcp -j NAIVEPROXY
```
```bash
sudo chmod +x /etc/naiveproxy/network.sh
```
#### rc.local startup
rc.local script:
```bash
sudo vi /etc/rc.local
```
```bash
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

/etc/naiveproxy/network.sh
 
exit 0
```
```bash
sudo chmod +x /etc/rc.local
```
rc.local service config:
```bash
sudo vi /etc/systemd/system/rc-local.service
```
```bash
[Unit]
Description=/etc/rc.local
ConditionPathExists=/etc/rc.local
 
[Service]
Type=forking
ExecStart=/etc/rc.local start
TimeoutSec=0
StandardOutput=tty
RemainAfterExit=yes
SysVStartPriority=99
 
[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl enable rc-local
```
#### Supervisor
Install package:
```bash
sudo apt update && sudo apt install supervisor
```
Setup config file:
```bash
sudo vi /etc/supervisor/conf.d/supervisor.conf
```
```bash
[program:naive_client_1]
command=/usr/local/bin/naiveproxy /etc/naiveproxy/conf.d/1.json
autorestart=true
user=root

[program:naive_client_2]
command=/usr/local/bin/naiveproxy /etc/naiveproxy/conf.d/2.json
autorestart=true
user=root

[program:naive_client_3]
command=/usr/local/bin/naiveproxy /etc/naiveproxy/conf.d/3.json
autorestart=true
user=root
```
Restart the application and check status:
```bash
sudo systemctl restart supervisor
sudo systemctl status supervisor
```
#### HAProxy
Install package:
```bash
sudo apt install haproxy
```
Setup config file:
- IMPORTANT:
	- change server IP address and port to your own;
	- change login credential to HAProxy stats portal.
```bash
sudo vi /etc/haproxy/haproxy.cfg
```
```bash
global
	#nbproc  2
	#cpu-map  1 1
	#cpu-map  2 2
	log /dev/log local0 notice
	# log /dev/log	local0
	# log /dev/log	local1 notice
	chroot /var/lib/haproxy
	# stats socket /run/haproxy/admin.sock mode 660 level admin
	# stats timeout 30s
	user haproxy
	group haproxy
	daemon
	max-spread-checks 5000
	spread-checks 2
	maxconn 60000

defaults
	default-server inter 10s fastinter 2 downinter 30s rise 30 fall 1 minconn 100 maxconn 8000 maxqueue 100 resolve-prefer ipv4 slowstart 5m weight 100
	log global
	maxconn 50000
	mode tcp
	option tcpka
	option contstats
	option tcplog
	option logasap
	option dontlognull
	option dontlog-normal
	option log-separate-errors
	retries 1
	option redispatch
	timeout connect 1s
	timeout check 10s
	timeout client 120s
	timeout client-fin 100s
	timeout http-keep-alive 30s
	timeout queue 1m
	timeout server 120s
	timeout server-fin 100s
	timeout tarpit 5s
	timeout tunnel 5m

frontend naive_server_in
	bind 127.0.0.1:1443
	default_backend naive_server_out
backend naive_server_out
	option tcp-check
#   option allbackups
	option redispatch
	retry-on all-retryable-errors
	server s1 a.b.c.d:443 check weight 100
	server s2 e.f.g.h:50443 check backup weight 100

listen stats
        bind :2090
        mode http
        stats enable
        stats hide-version
        stats realm HAproxy-Statistics
        stats uri /haproxy_stats
        stats auth haproxy:B7ngSl0dxj5tOE4xWS38oskiQlGhrNQo
        stats refresh 3s
```
Restart the application and check status:
```bash
sudo systemctl restart haproxy
sudo systemctl status haproxy
```
#### Check
Restart the client machine and check if everything works.