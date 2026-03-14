---
title: "Automated Backups with Proxmox Backup Server"
date: 2026-02-07
draft: false
description: "Set up Proxmox Backup Server to automate incremental backups of your VMs and LXC containers with deduplication."
tags: ["proxmox", "backup", "pbs", "homelab", "storage"]
categories: ["Infrastructure"]
series: ["Home Network Build"]
showToc: true
---

## The Problem with Ad-hoc Backups

Proxmox VE has built-in backup functionality — you can snapshot a VM to a directory or NFS share on a schedule. But it stores full backups each time, space grows fast, and restoring requires the whole archive. Proxmox Backup Server (PBS) solves all three problems.

PBS is a dedicated backup server that:

- Stores backups with **client-side deduplication and compression** (typically 50–80% space savings)
- Does **incremental backups** — only changed chunks are uploaded
- Supports **instant verification** by recalculating checksums
- Has **pruning policies** — keep 7 daily, 4 weekly, 12 monthly backups automatically

## Architecture

PBS is a separate Debian-based appliance. It does not run inside Proxmox VE. You have a few options:

- **Dedicated machine** — ideal, but overkill for most homelabs
- **Second Proxmox node** — run PBS as a VM on a different node from what it's backing up
- **Same node VM** — convenient, but you lose backups if the node dies

I run PBS as a VM on my second Proxmox node (`pve-02`), backing up VMs from both nodes. The backup storage is a 2 TB HDD passed through to the PBS VM.

## Installing Proxmox Backup Server

Download the PBS ISO from the [Proxmox downloads page](https://www.proxmox.com/en/downloads). It installs like Proxmox VE — boot from USB, follow the wizard, give it a static IP.

The web UI runs on port **8007**: `https://pbs-ip:8007`

## Setting Up a Datastore

A datastore is where backups are stored. In the PBS UI:

1. Go to **Datastore -> Add Datastore**
2. Set a name (e.g., `main-backup`)
3. Set the path to your backup disk (e.g., `/mnt/backup-disk`)
4. Configure a **Prune Schedule** — I use: keep last 7 daily, 4 weekly, 3 monthly

PBS creates its chunk store structure in that directory automatically.

## Adding PBS to Proxmox VE

In the Proxmox VE web UI:

1. **Datacenter -> Storage -> Add -> Proxmox Backup Server**
2. Enter the PBS server IP, port 8007, your PBS username/password
3. Select the datastore
4. Set a fingerprint (copy from PBS: **Configuration -> Certificates -> Fingerprint**)

The PBS storage now appears as a backup target on your Proxmox nodes.

## Creating a Backup Job

In Proxmox VE:

1. **Datacenter -> Backup -> Add**
2. Choose your PBS storage
3. Select VMs/containers to back up (or "all")
4. Set a schedule — I run nightly at 2 AM
5. Set compression to `zstd` (best speed/ratio balance)
6. Enable **Send email on error** if you have SMTP configured

The first backup is a full transfer. All subsequent backups only upload changed data chunks.

## Monitoring Backup Jobs

In Proxmox VE, check **Datacenter -> Backup -> Job History**. In PBS, go to **Datastore -> main-backup -> Content** to see all stored backups with sizes and timestamps.

You can also verify backup integrity from PBS:

```bash
proxmox-backup-client verify <datastore> <backup-id>
```

## Restoring a VM

From Proxmox VE, find the VM in the resource tree, go to **Backup**, select a restore point, and click **Restore**. You can restore to any node in the cluster and rename the VM.

For granular file-level restores, PBS mounts the backup as a FUSE filesystem:

```bash
proxmox-backup-client mount <datastore> <backup-id> /mnt/restore
```

Then browse and copy individual files.

## Pruning and Retention

PBS prunes automatically according to your datastore schedule. You can also trigger a prune manually from the web UI or CLI:

```bash
proxmox-backup-manager prune --keep-daily 7 --keep-weekly 4 --keep-monthly 3 <datastore>
```

Pruning removes backup snapshots but the underlying chunks are only freed during garbage collection:

```bash
proxmox-backup-manager garbage-collection start <datastore>
```

Schedule weekly GC runs to keep disk usage accurate.

## Offsite Replication

PBS supports remote sync — you can replicate a datastore to a second PBS instance at a different location (friend's house, VPS). In the PBS UI: **Sync Jobs -> Add**.

This gives you offsite backups without needing a cloud service. Both endpoints deduplicate independently, so only unique chunks are transferred.

## Storage Sizing

A rough formula: assume your VMs total X GB of used disk, expect 60% compression, and keep N snapshots with ~5% daily change rate:

```
Storage needed = X * 0.4 * (1 + N * 0.05)
```

For 500 GB of VMs with 30 snapshots: `500 * 0.4 * 2.5 = 500 GB` of backup storage. Your actual results will vary depending on data type — highly compressible text/configs will do much better.
