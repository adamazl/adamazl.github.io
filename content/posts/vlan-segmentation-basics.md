---
title: "VLAN Segmentation: Isolating Your IoT Devices"
date: 2026-03-05
draft: false
description: "A practical guide to segmenting your home network with VLANs to keep IoT devices away from your trusted machines."
tags: ["networking", "vlan", "security", "iot", "firewall"]
categories: ["Networking"]
series: ["Home Network Build"]
showToc: true
---

## Why Segment Your Network?

The average home today has dozens of connected devices — smart bulbs, cameras, thermostats,
TVs. Most of these devices have poor security track records: default credentials, infrequent
firmware updates, and sometimes outright malicious firmware from vendors.

Putting them on the same flat network as your laptop and NAS is an unnecessary risk. VLANs fix
this.

## What Is a VLAN?

A **Virtual LAN (VLAN)** is a logical partition of a physical network. Devices on different VLANs
cannot communicate with each other unless you explicitly allow it through firewall rules — even if
they share the same physical switch.

## VLAN Design for a Home Network

A reasonable starting layout:

| VLAN ID | Name | Purpose |
|---------|------|---------|
| 1 | Default | Avoid using this one for anything important |
| 10 | Trusted | Laptops, desktops, phones you own |
| 20 | IoT | Smart home devices, TVs, cameras |
| 30 | Guest | Wi-Fi for visitors |
| 40 | Servers | NAS, self-hosted services |
| 99 | Management | Switch/AP management interfaces |

## Prerequisites

- A **managed switch** that supports 802.1Q tagging (e.g., Netgear GS308E, TP-Link TL-SG108E)
- A **router/firewall** that supports VLANs (OPNsense, pfSense, Unifi, MikroTik)
- A **wireless access point** that can broadcast multiple SSIDs with VLAN tagging

## Basic Firewall Rules

The golden rule: **deny by default, allow explicitly**.

For the IoT VLAN, you typically want:

```text
# IoT VLAN firewall rules (in order)
BLOCK  IoT → Trusted   (IoT cannot reach your laptops/NAS)
BLOCK  IoT → Servers   (IoT cannot reach internal services)
ALLOW  IoT → Internet  (IoT can reach the outside world)
BLOCK  IoT → *         (catch-all deny)
```

## Next Steps

In the next post, we'll walk through the actual configuration steps in OPNsense — creating the
VLAN interfaces, setting up DHCP per VLAN, and writing the firewall rules.
