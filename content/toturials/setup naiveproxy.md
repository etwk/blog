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
### Linux Server Optimize
[Debian server optimize](../toturials/Debian%20server%20optimize.md)
### Naïve Server
todo
