---
title: Access your Jellyfin from anywhere with tailscale and external VPS
date: 2024-04-01 20:00:00 +0100
categories: [Homelab, truenas]
tags: [jellyfin, truenas, tailscale, vps]
math: false
mermaid: false
---
# Introduction
Simple guide to show you a way of accessing your Jellyfin server using an external VPS with taiscale wireguard tunnel for as little as 10$/year.

## Advantages
* no public IP address needed
* bypasing CGNAT of your IPS provider
* encrypted traffic (IPS can't collect data and is less likely to throttle you)
* total anonymity (with use of Duck DNS)

> This guide was tested using jellyfin running on the truenas, but the principle can be applied on any platform.
{: .prompt-info }
 
# Quick FAQ
## Why not just port-forward to proxy?
* The port-forward may not be possible if you don't have public IP address or your IPS is using CGNAT.

## Why not cloudflare tunnel?
* Streaming data from plex/jellyfin violated Cloudflare's terms of service.
* Cloudflare also [inspects](https://developers.cloudflare.com/analytics/network-analytics/reference/data-collection/) your traffic, which may not really be a good combinaton with you streaming "all your bought movies"
* By default the bandwitch is too small for movie streaming ** 

** this can be overcomed with some tinkering [A] (https://www.cloudflare.com/learning/cdn/what-is-a-cdn/) [B] (https://selfhosters.net/docker/plex/cloudflare/)

## Why not just Wireguard, netbird, twingate?
* While all of these VPN providers use wireguard as their tunnel technology, wireguard on it's own can be difficult to start with for beginners.
* Netbird, twingate are good alternatives to tailscale. In the end it's all about the prefrence and what you are already using.


# How to
# Prereq
### Choosing VPN provider
You can find best deals using site [lowendstock.com](lowendstock.com). 
While these providers can be on the more "suspicious" site of things, when you compare the pricing it's a nobrainer for VPN proxy like this. 
You can find VPS for as little as 10USD/year on Black fridays/Chinese New Year.

#### Tips
* use KVM platform
* choose IPV4 = 1 (we want the public IP address)
* probably a good thing to avoid Germany and Russia as an country origin of VPS provider


### Choosing Domain
We will need public domain for https traffic and ease of use. There are few options to go with

#### Use Duck DNS
Probably the easiest and the most annonymous way is using a [Duck DNS](https://www.duckdns.org/) 
If you are not familiar with DuckDns, it's simple, secure way of using public DNS [which will be yours](https://www.duckdns.org/faqs.jsp) for free. 
Sound almost to good to be true right?  Well, no one hre garrantee you anything, but it's seems to be well established in the community by now.
With duck dns, you can create account and domain without any back-tracing information about you. 


#### Buy/Use an existing domain 
Use cloudflare or some other domain provider and buy something cheap for few USD/year.
If you already own a domain, just use that. In the end we only need one A record.

