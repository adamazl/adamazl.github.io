---
title: "Building a Home NAS with TrueNAS Scale"
date: 2026-01-10
draft: false
description: "Set up TrueNAS Scale for a home NAS with ZFS storage, SMB shares, and app containers."
tags: ["truenas", "nas", "zfs", "storage", "homelab"]
categories: ["Storage"]
showToc: true
---

## Why TrueNAS Scale?

TrueNAS Scale is a Debian-based NAS operating system built around ZFS. The "Scale" version adds Linux containers (Docker/Kubernetes) on top of the traditional NAS features, so your storage box can also run apps like Jellyfin or Nextcloud alongside your shares.

It's free, open-source, and the ZFS integration is first-class. Data integrity checks, snapshots, and replication are all built in and accessible through the web UI.

## Hardware Recommendations

ZFS is memory-hungry and loves ECC RAM. For a home NAS:

- **CPU:** Anything modern with AES-NI (for encryption). Even an older Xeon handles it fine.
- **RAM:** 8 GB minimum; 16 GB if you run apps. ZFS uses RAM for ARC (caching). More RAM = better read performance.
- **OS disk:** A small SSD (32–120 GB). Mirrored is ideal. Do not put ZFS pools on the OS disk.
- **Data drives:** Match sizes within a vdev for best space efficiency. Helium-filled drives are quieter and use less power.
- **HBA:** If you have more than 4 drives, a proper HBA (IT mode) is cleaner than consumer SATA controllers.

## Installation

Download the TrueNAS Scale ISO, write it to USB, boot and install. The installer is straightforward — select your OS disk and set an admin password.

Once installed, access the web UI at the IP shown on the console, port 80/443.

## Pool Creation

This is the most important step. Plan your pool layout before creating it — migrating later is painful.

### Common Layouts

| Drives | Layout | Notes |
|--------|--------|-------|
| 2 | Mirror | Full redundancy, 50% usable |
| 4 | Mirror x2 (striped mirrors) | Better performance than RAIDZ2 with 4 drives |
| 4–6 | RAIDZ2 | 2 drive fault tolerance |
| 6+ | RAIDZ3 or striped mirrors | Enterprise-grade redundancy |

For a home NAS with 4 drives, I recommend **2x mirrored vdevs** (striped mirrors, or "RAID 10"). You can easily add more mirror vdevs later to expand. RAIDZ cannot be expanded without replacing all drives.

In TrueNAS: **Storage -> Create Pool**

1. Name the pool (e.g., `tank`)
2. Drag drives into the layout
3. Select pool type
4. Enable **Encryption** if desired (AES-256, uses your admin password to unlock on boot)

## Creating Datasets

Datasets are ZFS filesystems within a pool. Create separate datasets per purpose — don't dump everything in the pool root.

```
tank/
  media/      # Jellyfin library
  backups/    # VM backups from Proxmox
  documents/  # Personal files
  nextcloud/  # Nextcloud data
```

In TrueNAS: **Datasets -> Add Dataset**

Set compression to `lz4` (default, good balance of CPU/ratio) or `zstd` for more compression at slight CPU cost.

## SMB Shares

For Windows/macOS file sharing:

1. **Shares -> Windows (SMB) -> Add**
2. Path: select your dataset
3. Name: the share name (e.g., `media`)
4. Enable **Time Machine** if you want Macs to back up to it

Create TrueNAS local users and map them to SMB share permissions under **Credentials -> Local Users**.

## NFS Shares

For Linux/Proxmox:

1. **Shares -> Unix (NFS) -> Add**
2. Path: your dataset
3. Networks: restrict to your LAN CIDR (e.g., `192.168.1.0/24`)
4. Maproot User: root (or a specific user)

On Proxmox, add the NFS storage under **Datacenter -> Storage -> Add -> NFS**.

## Snapshots and Auto-Snapshots

ZFS snapshots are instant and space-efficient — they only store changed blocks. Set up a snapshot schedule:

**Data Protection -> Periodic Snapshot Tasks -> Add**

A sensible policy:
- Hourly snapshots, keep 24
- Daily snapshots, keep 30
- Weekly snapshots, keep 8

Snapshots protect against accidental deletion and ransomware (snapshots are read-only). They're not a backup — you still need offsite copies.

## Scrubs

A scrub reads every block in the pool and verifies checksums, correcting any data that has silently corrupted. TrueNAS schedules monthly scrubs by default. Don't disable these.

Scrubs run in the background with minimal performance impact. Check scrub results under **Storage -> Pool Status** — a healthy pool shows zero errors after a scrub.

## Apps (Containers)

TrueNAS Scale includes an apps section (based on Helm/Kubernetes). You can deploy Nextcloud, Jellyfin, Plex, and dozens of other apps directly from the UI without managing Docker yourself.

**Apps -> Discover Apps** and search for what you need. Apps store their data in datasets you specify, keeping your data separate from the container.

For more control, TrueNAS also supports custom app deployment via Docker Compose through the community TrueCharts catalog.

## iSCSI for Block Storage

TrueNAS supports iSCSI for presenting block devices to hosts. This is useful for:

- Presenting storage to Proxmox VMs as a virtual disk
- Giving a Windows VM a raw disk for better NTFS performance

**Shares -> iSCSI** — create a zvol (ZFS volume), then create a target, extent, and map them together.

## Monitoring

TrueNAS has built-in reporting under **Reporting** — CPU, memory, disk activity, network. It also supports Graphite and Netdata for external monitoring.

Set up email alerts under **System -> Alert Settings** to get notified of drive failures, pool degradation, and scrub results.
