---
layout: post
title:  "Exploring Dual WAN Solutions on a Budget"
date:   2024-11-18 22:44:00 +0800
categories: jekyll update
---
# Exploring Dual WAN Solutions on a Budget

The issue began when one day, the ISP plan that I have been using finally expired. My internet got cut off, and the WAN light on my router began flashing red. The fallback? Tethering on my phone's hotspot, which worked well for a short-term emergency. However, the vast majority of my home systems—still connected to my router and its now non-functional WAN interface—were left without access to the wider internet.

This experience got me thinking: in the event that my ISP connection goes down[^1], is it possible to have my default gateway (i.e. my TP-link router) route traffic through my phone hotspot so that all devices connected to my router can still continue to access the internet to a certain extent[^2]?


## WAN-failover

Thankfully, relying on a secondary connection when the primary one fails is an established concept known as **WAN-failover** (or dual-WAN). The naming makes sense, as it involves failing over to a second WAN or ISP connection when the first one fails, ensuring continuous internet connectivity for all devices connected to the router.

Unfortunately, WAN-failover appears to be a feature only commonly found in enterprise-grade routers. On most home-based consumer routers, there is only a single WAN interface available. There is also no place in the router's web interface for configuring a WAN failover point.

This begs the question: where can I introduce my second internet connection (in this case, my phone's hotspot) so that my network remains connected to the wider internet?

## Possible approaches

### Approach 1: Re-route all traffic to another device connected to the router.

The ultimate goal of this is to have as little disruption to my existing network as possible. I have many devices. My family has many devices. Manually connecting each of them to my phone’s hotspot in the event of an outage would be an unwelcome hassle.


Instead, I want my TP_Link router to do the routing for me, to route traffic destined for the WAN to another internet facing device.

```
Before: home_devices <--> TP_Link router <--> WAN

Situation: home_devices <--> TP_Link router <-/ outage /-> WAN

Solution: home_devices <--> TP_Link router <-/ outage /-> WAN
                                    |
                                    +------> device/router <--> hotspot

```

Taking a look at my router's routing table, it is most certainly possible:

Network Dest | Subnet Mask | Gateway | Interface
-------------|-------------|---------|---------
0.0.0.0 | 0.0.0.0 | ISP's default gateway | WAN
ISP's subnet | ISP's subnet mask | 0.0.0.0 | WAN
192.168.0.0 | 255.255.255.0 | 0.0.0.0 | LAN

Currently, all external traffic is routed to ISP's default gateway via the WAN interface. However, if I could modify the routing table to look like this


Network Dest | Subnet Mask | Gateway | Interface
-------------|-------------|---------|---------
0.0.0.0 | 0.0.0.0 | 192.168.0.2 | LAN
ISP's subnet | ISP's subnet mask | 0.0.0.0 | WAN
192.168.0.0 | 255.255.255.0 | 0.0.0.0 | LAN

Then, all traffic destined for the internet would instead route to a local device at 192.168.0.2 via the LAN port. From there, the local device could forward these packets to the phone's hotspot.

We do end up with a double NAT, where the first NAT is our local device at 192.168.0.2, and the second NAT is at our phone. While not ideal, it’s workable.


Unfortunately, this plan hinges on the ability to modify the routing table on my TP-Link router. Sadly, I discovered that my router blocks any attempt to add routes destined for 0.0.0.0/0, ruling out this option entirely.

### Approach 2: Add a device between my TP-Link Router and the WAN.

On the WAN-side, we observe the current setup:

```
Before: home_devices  <--> TP_Link Router <--> WAN

After: home_devices  <--> TP_Link Router <--> A Router that supports dual WAN! <--> WAN1
                                                 |
                                                 +-------------> WAN2
```

This approach introduces a dual-WAN router between the TP-Link router and the WAN. The dual-WAN router would handle failover or load balancing between the primary WAN (ISP) and the secondary WAN (hotspot). My TP-Link router could then act as a simple access point, connecting all wired and wireless devices to this custom router.

The exciting part is that I don’t need to purchase a separate dual-WAN device. My Intel N100 NUC has dual 2.5G NICs, making it a great candidate for acting as the router between my TP-Link router and the WAN. With Proxmox installed, I could virtualize a dual-WAN-capable system like pfSense or OPNsense, providing advanced routing and failover capabilities typically found in enterprise-grade equipment.

Alternatively, I could use Linux-based tools (iptables, ip route, etc.) to manually configure dual-WAN routing. Either way, this setup would offer me complete control over how failover and load balancing are handled.

By switching my TP-Link router to AP mode, I would simplify the network configuration, avoid double NAT, and ensure seamless connectivity for all devices in the event of a failover.

## Conclusion for Part 2

Theoretically, this DIY setup appears both feasible and cost-effective. With the Intel N100 acting as a custom router, I could introduce dual WAN functionality to my home network without overhauling my existing devices. However, the devil is in the details. Will Proxmox, pfSense, or a similar tool handle failover as seamlessly as I hope? Can the NUC manage the routing workload effectively, and will it perform as well in practice as it does in theory?

This DIY setup seems promising, at least in theory. If it works, I’ll have found a practical solution to the problem I initially set out to solve: ensuring internet connectivity for my home network during outages. The next step is to test and see if it holds up in practice. Stay tuned!

-------------------
[^1]: happens fairly frequently in SG ngl

[^2]: i say "certain extent" because when we switch over to a different ISP / WAN provider, our external IP changes. This can have several noticeable side-effects. For one, I can imagine disconnecting from a zoom-call since our IP has changed, and would necessitate another renegotiation of websocket or whatever protocol with our new external IP to join back the call.
