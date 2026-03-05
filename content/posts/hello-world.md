---
title: "Hello, Homelab"
date: 2026-03-05
draft: false
description: "Kicking off the blog with an overview of my homelab goals and current setup."
tags: ["homelab", "intro", "self-hosting"]
categories: ["General"]
series: ["Home Network Build"]
showToc: true
---

## Why a Homelab?

Every homelab starts somewhere. Mine started with the frustration of paying for cloud services I
could run myself, and curiosity about what actually happens when packets travel across a network.

## Current Goals

1. **Full network segmentation** — IoT devices on their own VLAN, completely isolated from the
   trusted network
2. **Self-hosted DNS** with ad-blocking (Pi-hole / AdGuard Home)
3. **A proper NAS** for media, backups, and general storage
4. **Monitoring stack** — Prometheus + Grafana so nothing breaks silently

## Hardware on the Bench

| Device | Role |
|--------|------|
| TP-Link ER605 | Router / gateway |
| TP-Link SG2008P | Managed PoE switch |
| TP-Link OC200 | Omada hardware controller |
| TP-Link EAP245 | Wi-Fi access point |
| Proxmox VE × 2 | Compute nodes (virtualisation) |
| Proxmox PBS | Backup server |

## What's Coming Next

The first series of posts will cover setting up a flat-to-segmented network from scratch —
starting with the router/firewall choice all the way through VLAN tagging and inter-VLAN routing
rules.

Stay tuned.
