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
- white-list mode: all tcp access to IP addresses outside China goes through naiveproxy, others go direct.
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
### [Debian Server Optimize](../toturials/Debian%20server%20optimize.md)

### Naive Server
Please also check official documents for updated information: https://github.com/klzgrad/naiveproxy

#### [Obtain SSL Certificate](.//SSL%20certificate.md)

#### Update system settings
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

#### Setup Naive Server

Install go: https://golang.org/doc/install

Install caddy with Naive:
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

:443, abc.someDomain.com
tls "/etc/ssl/cert.d/abc.someDomain.com/fullchain.pem" "/etc/ssl/cert.d/abc.someDomain.com/privkey.pem"
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