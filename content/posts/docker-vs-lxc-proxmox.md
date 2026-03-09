---
title: "Docker vs LXC Containers in Proxmox: When to Use Each"
date: 2026-03-01
draft: false
description: "Understanding the trade-offs between Docker containers and LXC containers in a Proxmox homelab."
tags: ["proxmox", "docker", "lxc", "containers", "homelab", "virtualisation"]
categories: ["Infrastructure"]
showToc: true
---

## The Confusion

New Proxmox users often ask: should I run Docker inside a VM, Docker inside an LXC container, or use Proxmox's native LXC containers directly? The answer depends on what you're running and what you value. This post breaks down the trade-offs.

## What Is LXC?

LXC (Linux Containers) is an OS-level virtualisation technology. LXC containers share the host kernel but have their own filesystem, process tree, network stack, and user namespace. They boot like lightweight VMs — you run `pct start 101` and get a full Linux environment in under a second.

Proxmox manages LXC directly. Each container gets an entry in the Proxmox resource tree, shows up in graphs, and can be managed with the same tools as VMs.

## What Is Docker?

Docker is an application container runtime. Where LXC gives you "a Linux system", Docker gives you "a single application and its dependencies". Docker images are layered, immutable, and portable. The Docker daemon handles networking, storage, and lifecycle.

Docker is not a hypervisor or an OS virtualiser — it's a packaging and runtime standard for applications.

## Key Differences

| Property | LXC | Docker |
|----------|-----|--------|
| Unit | OS environment | Application |
| Management | Proxmox UI/CLI | docker CLI / Compose |
| Overhead | ~1–5 MB RAM | ~5–50 MB RAM |
| Startup | Under 1 second | 1–5 seconds |
| Portability | Host-dependent | Highly portable |
| Image ecosystem | DIY | Huge (Docker Hub) |
| Networking | Proxmox-managed | Bridge/overlay/host |
| Storage | Proxmox storage | Volumes / bind mounts |

## When to Use LXC

**LXC is the right choice when:**

- You want something that behaves like a VM but lighter
- You're running a service that isn't available as a Docker image
- You want Proxmox to manage the whole lifecycle (backups, snapshots, migration)
- You need a persistent environment you'll SSH into regularly (e.g., Pi-hole, WireGuard, monitoring stack)
- You're running privileged operations that are awkward in Docker (e.g., running your own Docker daemon)

**My LXC containers:**

- Pi-hole (DNS server — single long-lived service, Proxmox backup covers it)
- WireGuard VPN (network-adjacent, easier with host networking)
- Proxmox Backup Server client agent
- Home Assistant (benefits from Proxmox backup, needs USB passthrough)

## When to Use Docker

**Docker is the right choice when:**

- A project only ships Docker images (no bare-metal install supported)
- You want to run many services and manage them with Compose
- You want easy upgrades: `docker compose pull && docker compose up -d`
- Portability matters — you might move services between hosts
- You need a complex multi-container setup (app + database + cache)

**My Docker containers (inside a Debian LXC):**

- Grafana + Prometheus + node_exporter (multi-container, Compose keeps it tidy)
- Nginx Proxy Manager
- Nextcloud (app + PostgreSQL + Redis in one Compose file)
- Vaultwarden (Bitwarden server)
- Paperless-ngx (document management)

## Running Docker Inside LXC

This is my preferred pattern. I run a single "docker host" LXC container, install Docker in it, and run all my Docker Compose stacks there. Proxmox backs up the entire LXC, including all Docker data.

To enable Docker in an LXC container, the container must be **unprivileged** with some extra kernel features enabled. In Proxmox, edit the container config at `/etc/pve/lxc/ID.conf`:

```
features: keyctl=1,nesting=1
```

Then install Docker normally inside the container:

```bash
curl -fsSL https://get.docker.com | sh
```

Note: you cannot run Docker inside a privileged LXC container on Proxmox without additional work — nesting is the cleaner approach.

## Full VMs vs Both

Use a full KVM VM when:

- You need a Windows guest
- The workload needs its own kernel (custom kernel modules, eBPF, kernel development)
- You need strong security isolation (multi-tenant or untrusted workloads)
- You need PCIe passthrough (GPU, NIC, HBA)

VMs have the most overhead but the strongest isolation and full hardware emulation support.

## A Practical Homelab Layout

Here's how I structure things:

```
pve-01
├── VM: TrueNAS Scale (NAS — needs ZFS, dedicated disks via HBA passthrough)
├── VM: Windows 11 (gaming/work)
├── LXC: pihole (DNS)
├── LXC: wireguard (VPN)
└── LXC: docker-host (runs all Compose stacks)

pve-02
├── VM: Proxmox Backup Server
├── LXC: home-assistant (with USB Zigbee dongle passthrough)
└── LXC: monitoring (node_exporter, logs)
```

VMs get their own kernel and full isolation. LXC handles services that benefit from Proxmox management. Docker (inside LXC) handles the long tail of containerised apps.

## Snapshots and Backups

One advantage of LXC over running Docker on bare metal: Proxmox can snapshot and backup the entire LXC container, including all Docker volumes and configurations. You don't need to maintain separate backup scripts for each Docker application.

The trade-off: LXC containers are tied to the Proxmox host's kernel version. If a workload needs a specific kernel feature, use a VM instead.
