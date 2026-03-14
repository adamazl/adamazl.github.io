---
title: "Self-Hosted VPN with WireGuard"
date: 2025-11-01
draft: false
description: "Set up a WireGuard VPN server on your homelab so you can securely access your network from anywhere."
tags: ["wireguard", "vpn", "networking", "security", "linux"]
categories: ["Networking"]
series: ["Home Network Build"]
showToc: true
---

## Why Self-Host a VPN?

A self-hosted VPN gives you a secure tunnel back into your home network when you're away. Unlike commercial VPN services (which are for hiding traffic from your ISP), this is about remote access — connecting to your NAS, home automation, internal dashboards, or development environment from a coffee shop or hotel.

WireGuard is the right choice today. It's built into the Linux kernel, uses modern cryptography (ChaCha20, Curve25519), and has a drastically smaller codebase than OpenVPN (~4,000 lines vs ~400,000). Handshakes complete in milliseconds. Battery drain on mobile is noticeably lower.

## Prerequisites

- A Linux machine with a public IP (or a port forwarded through your router)
- A static IP or dynamic DNS name for your home connection
- Root/sudo access

I run WireGuard in a Debian 12 LXC container on Proxmox.

## Installing WireGuard

On Debian/Ubuntu:

```bash
apt update && apt install wireguard -y
```

WireGuard is in the mainline kernel since 5.6, so there's no kernel module to install separately on modern systems.

## Generating Key Pairs

Every WireGuard peer (server and each client) needs a private/public key pair.

**Server keys:**

```bash
cd /etc/wireguard
wg genkey | tee server_private.key | wg pubkey > server_public.key
chmod 600 server_private.key
```

**Client keys (generate one per device):**

```bash
wg genkey | tee client1_private.key | wg pubkey > client1_public.key
```

## Server Configuration

Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.10.0.1/24
ListenPort = 51820
PrivateKey = <contents of server_private.key>

# Enable IP forwarding and NAT so clients can reach your LAN
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# Laptop
PublicKey = <contents of client1_public.key>
AllowedIPs = 10.10.0.2/32
```

Replace `eth0` with your actual outbound interface (`ip route | grep default` shows it).

## Enable IP Forwarding

```bash
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

## Start WireGuard

```bash
systemctl enable --now wg-quick@wg0
```

Check the interface is up:

```bash
wg show
```

## Client Configuration

Create a config file for your laptop (`client1.conf`):

```ini
[Interface]
Address = 10.10.0.2/24
PrivateKey = <contents of client1_private.key>
DNS = 192.168.1.53  # your Pi-hole IP, optional

[Peer]
PublicKey = <contents of server_public.key>
Endpoint = your-home.dyndns.com:51820
AllowedIPs = 192.168.1.0/24, 10.10.0.0/24
PersistentKeepalive = 25
```

**AllowedIPs controls your routing:**

- `0.0.0.0/0` — route all traffic through the VPN (full tunnel)
- `192.168.1.0/24, 10.10.0.0/24` — only route home network traffic (split tunnel)

Split tunnel is usually what you want for homelab access — your regular browsing stays on the local internet, only homelab-bound traffic goes through the tunnel.

## Port Forwarding

On your router, forward UDP port `51820` to your WireGuard server's LAN IP. On OPNsense: **Firewall -> NAT -> Port Forward**.

## Dynamic DNS

If your ISP gives you a dynamic IP (most residential connections), use a DDNS service so your endpoint address stays consistent. Popular options:

- **Cloudflare** — free, update via API
- **DuckDNS** — free, simple
- **No-IP** — free tier available

Most routers have built-in DDNS clients. OPNsense supports Cloudflare DDNS under **Services -> Dynamic DNS**.

## Adding More Clients

For each new device:

1. Generate a new key pair on the server
2. Add a `[Peer]` block to `wg0.conf` with a unique `AllowedIPs` address (10.10.0.3/32, etc.)
3. Reload: `wg syncconf wg0 <(wg-quick strip wg0)` (no downtime)
4. Create the client config file and import it

For mobile clients, use qrencode to generate a QR code the WireGuard app can scan:

```bash
apt install qrencode
qrencode -t ansiutf8 < client2.conf
```

## Revoking Access

Remove the peer's `[Peer]` block from `wg0.conf` and reload. Their key is immediately invalid — no certificate revocation lists needed.

## Monitoring

```bash
wg show  # live peer status, handshake times, data transfer
```

A peer with no recent handshake either hasn't connected yet or has a configuration issue. `PersistentKeepalive = 25` keeps the tunnel alive through NAT.
