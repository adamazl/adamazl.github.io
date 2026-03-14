---
title: "Network-Wide Ad Blocking with Pi-hole"
date: 2025-10-18
draft: false
description: "How to set up Pi-hole as your home network's DNS server for ad blocking, custom local DNS, and visibility into what your devices are actually doing."
tags: ["pihole", "dns", "networking", "raspberry-pi", "homelab"]
categories: ["Networking"]
series: ["Home Network Build"]
showToc: true
---

## Why Pi-hole

Most ad blockers work at the browser level. Pi-hole works at the DNS level, which means it blocks ads for every device on your network — smart TVs, phones, game consoles, IoT devices — without installing anything on them. It works by acting as the DNS resolver for your LAN and returning `0.0.0.0` for known ad and tracking domains instead of the real IP.

The side effect is that you also get a full picture of every DNS query every device makes, which is genuinely eye-opening. You will quickly discover that your smart TV is phoning home every few minutes.

## Hardware Options

Pi-hole runs on almost anything. Common choices:

- **Raspberry Pi** (Zero 2W, 3, 4, or 5) — the classic. A Pi Zero 2W at ~$15 is plenty for DNS.
- **LXC container in Proxmox** — my preferred approach. Zero additional hardware, easy to snapshot.
- **Docker container** — works fine, but the LXC approach gives a cleaner network stack.
- **Any spare Linux box or VM** — it's just a script.

For a production homelab, run two Pi-hole instances so you have redundancy. More on that below.

## Installing Pi-hole in an LXC Container

In Proxmox, create a new LXC container. I use the Debian 12 template.

**Container settings:**
- 1 vCPU, 512 MB RAM (256 MB works but gives no headroom)
- 4 GB disk
- Static IP — this is important. Your DNS server cannot have a DHCP address.

SSH into the container and run the official installer:

```bash
curl -sSL https://install.pi-hole.net | bash
```

The installer is interactive. Key choices to make:

1. **Upstream DNS provider** — I use Cloudflare (`1.1.1.1`) or Quad9 (`9.9.9.9`). You can change this any time.
2. **Blocklists** — accept the default StevenBlack list for now. You can add more later.
3. **Web interface** — yes, install it. It's the main way you'll interact with Pi-hole.
4. **Static IP** — confirm the static IP you configured at the container level.

At the end of the install, it will print the admin password. Save it.

Access the web interface at `http://<pihole-ip>/admin`.

## Pointing Your Network at Pi-hole

The easiest way to deploy Pi-hole is to configure your router to hand out the Pi-hole IP as the DNS server via DHCP. Every device that renews its DHCP lease will then use Pi-hole automatically.

In most router firmware (and in Proxmox's built-in DHCP, or your OPNsense/pfSense setup):

```
DHCP DNS Server 1: 192.168.1.2   # your Pi-hole IP
DHCP DNS Server 2: 1.1.1.1       # fallback, optional
```

If you set a fallback DNS, devices will bypass Pi-hole when it's down — which defeats the blocking but keeps the internet working. Your call on the trade-off.

For clients that don't respect DHCP DNS (some smart TVs hardcode `8.8.8.8`), you can intercept and redirect DNS at the firewall level with a NAT rule. In OPNsense:

```
Firewall > NAT > Port Forward
Protocol: TCP/UDP
Destination port: 53
Redirect target: 192.168.1.2 port 53
```

This catches any DNS query leaving your network and redirects it to Pi-hole regardless of what the client has configured.

## Adding Blocklists

The default list is solid but conservative. In the Pi-hole web UI, go to **Adlists** and add additional sources. Well-maintained lists:

- `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts` — the default, good general coverage
- `https://blocklistproject.github.io/Lists/ads.txt` — aggressive ad blocking
- `https://blocklistproject.github.io/Lists/tracking.txt` — tracking domains
- `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/multi.txt` — Hagezi Multi, excellent coverage

After adding new lists, go to **Tools > Update Gravity** (or run `pihole -g` from the CLI) to pull and compile the lists.

```bash
pihole -g
```

My current gravity database has around 1.2 million blocked domains. Query time is still sub-millisecond because it's an SQLite lookup.

## Local DNS Records

This is an underused feature. Pi-hole can serve local DNS records, so instead of bookmarking `192.168.1.10` you can use `proxmox.home.lan`.

Go to **Local DNS > DNS Records** and add entries:

| Domain | IP |
|--------|-----|
| proxmox.home.lan | 192.168.1.10 |
| nas.home.lan | 192.168.1.20 |
| pihole.home.lan | 192.168.1.2 |

You can also add CNAME records under **Local DNS > CNAME Records**, which is useful when you have multiple services behind a reverse proxy — they all CNAME to `proxy.home.lan` which resolves to the proxy's IP.

## Whitelisting and Troubleshooting

Pi-hole will occasionally block something it shouldn't. The query log (in the web UI under **Query Log**) shows every request in real time. When something breaks:

1. Check the query log and filter by the device's IP
2. Find the blocked domain
3. Add it to the whitelist: **Whitelist > Add domain**

From the CLI:

```bash
pihole whitelist example.com
pihole blacklist badtracker.com
pihole tail          # live log tail
pihole status        # service status
pihole restartdns    # restart without full reboot
```

## Running Two Pi-hole Instances

Single points of failure are bad when DNS is involved. If Pi-hole goes down, nothing on your network can resolve hostnames.

Options:

**Option 1 — Two independent Pi-holes.** Set up a second instance and configure DHCP to hand out both IPs as DNS servers. Simple, but the two blocklists/whitelists drift out of sync over time.

**Option 2 — Gravity Sync.** An open-source tool that keeps two Pi-hole instances in sync (blocklists, whitelists, custom DNS, everything).

```bash
# Install on both Pi-holes
curl -sSL https://raw.githubusercontent.com/vmstan/gravity-sync/master/install.sh | bash
```

After setup, run `gravity-sync push` from the primary to replicate to the secondary. You can cron this:

```bash
*/30 * * * * /usr/local/bin/gravity-sync push > /dev/null 2>&1
```

With two instances synced, point DHCP at both. Redundancy solved.

## What to Expect

After a day of running Pi-hole, check the dashboard. On a typical home network you should see 10–30% of queries blocked. If it's under 5%, your blocklists might not have updated. If it's over 50%, you might be blocking too aggressively and causing breakage.

The per-client view is the most useful part — you can see exactly which device is the chattiest and what it's talking to. Smart home devices in particular tend to generate a surprising amount of traffic.

Pi-hole is one of those tools that, once installed, you forget about — it just runs. The value is in the initial visibility it gives you into your network.
