---
title: "Setting Up OPNsense as Your Home Firewall"
date: 2026-02-25
draft: false
description: "Replace your ISP router with OPNsense for a proper firewall, VLAN support, and full traffic visibility."
tags: ["opnsense", "firewall", "networking", "homelab", "security"]
categories: ["Networking"]
series: ["Home Network Build"]
showToc: true
---

## Why Replace Your ISP Router?

ISP-provided routers are designed to be cheap and manageable by support staff, not to give you control. They have opaque firmware, rarely get security updates, and have none of the features a proper firewall offers: VLAN support, traffic shaping, IDS/IPS, meaningful logs, VPN server, DNS over TLS.

OPNsense is a FreeBSD-based firewall/router that runs on commodity x86 hardware. It's fully open-source (forked from pfSense in 2015), actively maintained, and has a polished web UI.

## Hardware

You need a machine with at least two NICs — one for WAN, one for LAN. More NICs give you dedicated interfaces for VLANs or DMZ segments.

Good options:

| Platform | Notes |
|----------|-------|
| Protectli Vault FW4B | 4-port Intel NIC, fanless, 8 GB RAM — purpose-built |
| Topton/Cwwk mini PCs | Cheap Chinese mini PCs with 4–6 Intel i225/i226 NICs |
| Old thin client (HP T620+) | Add a dual-port NIC via PCIe |
| Proxmox VM with SR-IOV | VirtIO NICs, good for testing |

I run OPNsense on a Topton N6005 mini PC with 4 Intel i226-V NICs. Low power (~10W), fanless, handles gigabit with IDS enabled.

## Installation

1. Download the AMD64 DVD installer from [opnsense.org](https://opnsense.org/download/)
2. Write to USB: `dd if=OPNsense-*.iso of=/dev/sdX bs=4M status=progress`
3. Boot the target machine from USB
4. Log in as `installer` / `opnsense`
5. Follow the installer — ZFS is recommended for reliability

After install, the console shows your WAN and LAN IPs. Access the web UI at `https://192.168.1.1` (default LAN).

## Initial Configuration

Default credentials: `root` / `opnsense`. Change immediately.

Run the **Setup Wizard** (it launches automatically on first login):

1. Set hostname, domain, DNS
2. Configure WAN — DHCP for most ISP connections; PPPoE if your ISP uses it
3. Configure LAN IP
4. Set timezone

## Interface Assignment

Go to **Interfaces -> Assignments** to review how physical NICs map to OPNsense interfaces. Assign additional NICs as OPT1, OPT2, etc., then rename them meaningfully (SERVERS, IOT, GUEST).

## VLANs

OPNsense handles VLANs at the software level, so one physical NIC can carry multiple VLANs to a managed switch.

**Interfaces -> Other Types -> VLAN -> Add:**

- **Parent Interface:** the NIC connected to your switch
- **VLAN Tag:** e.g., 20 for IoT
- **Description:** IoT

Assign the VLAN interface under **Interfaces -> Assignments**, enable it, set a static IP (gateway for that VLAN), and enable DHCP server for it.

Create firewall rules to control inter-VLAN traffic — by default VLANs cannot reach each other.

## Firewall Rules

OPNsense uses a top-down, first-match rules model per interface. Rules on the LAN interface control traffic entering from LAN.

Basic IoT VLAN rules:

1. **Pass** — IoT to port 53 (DNS to Pi-hole)
2. **Pass** — IoT to WAN (internet access)
3. **Block** — IoT to any (block all other traffic, including LAN)

Go to **Firewall -> Rules -> [your VLAN interface]** and add rules in order.

## DNS Over TLS

Send all DNS queries encrypted to a trusted resolver:

**Services -> Unbound DNS -> DNS over TLS:**

Add your preferred resolver:
- `1.1.1.1` / `one.one.one.one` (Cloudflare)
- `9.9.9.9` / `dns.quad9.net` (Quad9)

Enable **DNSSEC** validation. Under **General**, set the listening interface to LAN (and other interfaces that should use Unbound).

## IDS/IPS with Suricata

OPNsense includes Suricata for intrusion detection and prevention.

**Services -> Intrusion Detection -> Administration:**

1. Enable, set interface to WAN
2. **Download** rulesets — ET Open is free and good
3. Set mode to **IPS** (inline, drops matching traffic) or **IDS** (alerts only)
4. Click **Apply**

IPS mode adds a few milliseconds of latency but catches real threats. Start with IDS to see what it would block before switching.

## Traffic Shaping (QoS)

If you share your connection with others or have videoconferencing needs, traffic shaping prevents downloads from saturating the connection.

**Firewall -> Shaper -> Pipes/Queues/Rules:**

1. Create two pipes — one at your upload speed, one download
2. Create queues for high-priority (VoIP, video) and normal traffic
3. Add rules classifying traffic into queues by DSCP mark or port

## WireGuard Server

OPNsense has a WireGuard plugin:

**System -> Firmware -> Plugins -> os-wireguard**

After installing, go to **VPN -> WireGuard** to configure. The setup is the same as a standalone WireGuard server — generate key pairs, define peers, set allowed IPs. OPNsense automatically creates firewall rules for the tunnel interface.

## Monitoring and Logs

- **Interfaces -> Overview** — live per-interface traffic graphs
- **Firewall -> Log Files -> Live View** — real-time firewall log
- **Reporting -> Traffic** — historical bandwidth by interface
- **Dashboard widgets** — add gateway latency, firewall states, interface traffic

For external monitoring, OPNsense can export to InfluxDB or send NetFlow data to ntopng.

## Keeping OPNsense Updated

OPNsense releases updates frequently. Under **System -> Firmware -> Updates**, check for updates regularly. Major version upgrades are well-tested and usually straightforward. Always read the release notes before upgrading — occasionally an update requires manual intervention.
