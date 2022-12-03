---
title: "Cross the GFW"
tags: 
- network
- China
- censorship
- GFW
---

[The GFW](https://en.wikipedia.org/wiki/Great_Firewall) here refer to the censorship on the Internet between China and outside. It's relatively advanced for quite some years. In terms of website block, it's mainly working in these aspects:
1. Pollution or block of DNS resolution.
2. Block network access to IP addresses outside China.

Although this article here are mainly based on situations in China, many can be applied to other places as well.

## Guideline

### Aspects to think about
- Are the provider of VPN (or other related services) safe in itself?
- Consequences on possible expose your usage of VPN (or other similar services).
	- Social-engineering.
	- Some places like China might have laws that ban this kind of actions.
	- Server ports or IP addresses might get temporary or permanent ban by the GFW.
- Internet censorship status of edge server.
### Choose a provider
- That have good reputation, and you can trust. Be aware that some provider might will corporate with governments or even initiated by governments at the first place.
- Ensure they protect privacy by default, for example, no logging.

## Solution
### Deployment
- Establish network connection from end devices (such as mobile phone, laptop) to servers directly.
	- Might need to install client software the provider provided.
- Set up a transparent gateway in local network.
	- So that all devices within the network can use this gateway for unrestricted network access.
	- This can be set in router, computer, virtual machine, etc.
### Protocol
#### VPN
Some examples: OpenVPN, tinc, AnyConnect, IPSec, WireGuard

Advantage:
- Can route all traffic and avoid leak.
- Well-established ecology, lots of providers with lots of servers in different places.
Dis-advantage:
- Fingerprint easier to detect, easier for authority to obtain log of usage.
- Speed limited due to protocol, etc. 50Mbps is a good result for most them yet far lower than broadband connection in lots of places.
- Direct connection to servers outside China usually either not work or worked poorly.
#### Proxy
Some example: naiveproxy, V2Ray, Shadowsocks (also ShadowsocksR), Trojan

## Tech Introduction
### Transparent Gateway
#### DNS
##### Situation in China
DNS resolve to blocked websites such as Google are polluted in public DNS servers (such as 223.5.5.5); normal request (plain text UDP to port 53) send to public DNS servers outside China (such as 8.8.8.8) will be hijacked by the GFW and return wrong IP addresses.
##### Approach
A common approach is to set up a private DNS server, for example a network instance in one's own home.

- [Setup local DNS server].(toturial/setup%20local%20DNS%20server)

##### Connection

A local DNS server request DNS resolve results from public DNS servers within and outside China, connections to servers outside being protected:
- Establish VPN connection and route request out from VPN server outside China directly.
- Use UDP redirect feature provided by proxy software etc. to bypass restrictions on the GFW.
- DNS encryption, for example DNSCrypt.

##### Modes
There are different modes depending on how the DNS server acquire DNS resolution from which upper DNS server:
- (recommend) Whitelist mode: resolve these domains that are certainly has server within China by DNS servers in China, all others use servers outside China.
- Blacklist mode: resolve these domains that might have been blocked use DNS server outside China, all others use servers within China.

The reason behind different modes is that a lot of websites use CDN to provide faster access or redirect to regional site based on access location. Request from CDN servers nearby can usually improve speed a lot.

#### Network connection
##### General Situation
Connection from client to abroad will go through government controlled machine which can:
- Record metadata like source and destination IP addresses and port number, date time, header, etc.
- Wiretapping. They could see through plain text communication and decrypt some encrypted ones. They can't decrypt efficiently if encryption was done properly, but could store the data for future decryption.
- Block connection. This happens quite often between China and outside as a measure to block access to web services outside China.
##### Approach
Setup up VPN or proxy server in foreign country and route/redirect traffic out.

Might need to use servers in between to help route traffic in order to improve connectivity.

todo: list of tutorials
##### Connection
There are different parts to consider:
- For most website access, TCP redirect is enough.
- UDP
	- Needed for a lot of gaming services.
	- HTTP3 will use UDP, so we are seeing more and more UDP traffic happens with web surfing.
- VPN full routing
##### Modes
There are different modes depending on how do we route traffic out:
- (recommend) Route traffic to foreign IP addresses through foreign server; traffic to the current country go out directly or through a server within the current country.
	- If we have a lot of servers available in different countries, we can route traffic to different country each by servers within that country. This might help improve connectivity.
- Route all traffic through foreign server.