---
title: "ZFS for Homelabbers: Pools, Datasets, and Snapshots"
date: 2026-03-03
draft: false
description: "A practical guide to ZFS storage pools, datasets, compression, and snapshots for homelab NAS builds."
tags: ["zfs", "storage", "nas", "truenas", "linux"]
categories: ["Storage"]
showToc: true
---

## What Makes ZFS Different?

ZFS is not just a filesystem — it's a combined volume manager and filesystem. Everything from disk management to RAID to snapshots to checksumming is handled in one stack. This matters because:

- **Every block is checksummed.** Silent data corruption (bit rot) is detected and, with redundancy, automatically corrected.
- **Snapshots are instant and cheap.** A snapshot is just a pointer — it consumes no space until you delete data that the snapshot still references.
- **Copy-on-write semantics.** Writes never overwrite existing data. Torn writes (partial writes during power failure) cannot corrupt the filesystem.
- **Compression is transparent.** Enable it on a dataset and the CPU handles compression/decompression invisibly.

ZFS does not protect against drive failure any better than hardware RAID — it has the same RAIDZ fault tolerance. What it protects against is *silent* corruption, which hardware RAID controllers can silently propagate.

## Terminology

| Term | Meaning |
|------|---------|
| Pool | The top-level storage container. Spans one or more vdevs. |
| vdev | A virtual device — a group of drives in a RAID configuration |
| Dataset | A filesystem within a pool. Inherits pool settings, can override them. |
| Zvol | A block device within a pool. Used for iSCSI, swap, VMs. |
| ARC | Adaptive Replacement Cache — ZFS uses RAM for read caching |
| L2ARC | Second-level read cache on a fast SSD |
| SLOG | ZFS Intent Log — SSD for synchronous write acceleration |

## Pool Layouts

### Mirror

Two drives, exact copy. Can lose one drive and keep running.

```bash
zpool create tank mirror sda sdb
```

Can add more mirror vdevs to expand (striped mirrors = RAID 10):

```bash
zpool add tank mirror sdc sdd
```

### RAIDZ1 / RAIDZ2 / RAIDZ3

Variable stripe width with 1, 2, or 3 parity drives. RAIDZ cannot be expanded by adding drives later (without full replacement).

```bash
zpool create tank raidz2 sda sdb sdc sdd sde sdf  # 6 drives, 2 parity
```

Usable space: N - parity drives. With 6 drives RAIDZ2: 4 drives usable.

### Recommendation for Homelabs

- 2 drives: mirror
- 4 drives: 2x mirrors (striped), not RAIDZ1 (better performance and expandable)
- 6+ drives: RAIDZ2 or 3x mirrors

## Working with Datasets

Create a dataset:

```bash
zfs create tank/media
zfs create tank/backups
zfs create tank/documents
```

### Properties

Set compression on a dataset:

```bash
zfs set compression=lz4 tank/media        # fast, good ratio
zfs set compression=zstd tank/documents   # better compression, more CPU
```

Disable access time updates (reduces writes on busy datasets):

```bash
zfs set atime=off tank/media
```

Set a quota:

```bash
zfs set quota=500G tank/media
```

List dataset properties:

```bash
zfs get compression,atime,quota tank/media
```

### Inheritance

Child datasets inherit parent settings. Set compression on the pool root and all new datasets inherit it:

```bash
zfs set compression=lz4 tank
```

## Snapshots

Create a snapshot:

```bash
zfs snapshot tank/documents@2026-03-01
```

List snapshots:

```bash
zfs list -t snapshot
```

Roll back a dataset to a snapshot (destroys all changes since):

```bash
zfs rollback tank/documents@2026-03-01
```

Access snapshot contents without rolling back — ZFS exposes snapshots in a `.zfs/snapshot/` directory:

```bash
ls /tank/documents/.zfs/snapshot/2026-03-01/
```

Delete a snapshot:

```bash
zfs destroy tank/documents@2026-03-01
```

### Automated Snapshots

Use `zfs-auto-snapshot` or `sanoid` for policy-based automated snapshots.

Install sanoid on Debian:

```bash
apt install sanoid
```

Edit `/etc/sanoid/sanoid.conf`:

```ini
[tank/documents]
  use_template = production

[tank/media]
  use_template = media
  recursive = yes

[template_production]
  frequently = 0
  hourly = 24
  daily = 30
  weekly = 8
  monthly = 12
  autosnap = yes
  autoprune = yes

[template_media]
  daily = 7
  weekly = 4
  autosnap = yes
  autoprune = yes
```

Enable the systemd timer:

```bash
systemctl enable --now sanoid.timer
```

## Replication

Send a dataset to another pool (local or remote):

```bash
# Initial send
zfs send -R tank/documents@2026-03-01 | zfs receive backup/documents

# Incremental send (only changes since last snapshot)
zfs send -i @2026-03-01 tank/documents@2026-03-09 | zfs receive backup/documents
```

For automated incremental replication, use `syncoid` (part of the sanoid project):

```bash
syncoid tank/documents backup-server:backup/documents
```

This is how you build proper offsite backups — replicate to a machine at a different physical location.

## Scrubs

A scrub reads all data, verifies checksums, and repairs any corrupted blocks that can be corrected using redundancy:

```bash
zpool scrub tank
zpool status tank  # check progress and results
```

Schedule monthly scrubs via cron:

```bash
0 2 1 * * zpool scrub tank
```

A clean scrub output looks like:

```
errors: No known data errors
```

Any errors reported mean you have hardware issues to investigate. Replace failing drives promptly — resilver (rebuild) time on large drives with degraded performance can take days.

## Checking Pool Health

```bash
zpool status -v  # verbose, shows any errors per drive
zpool iostat -v  # per-vdev I/O statistics
```

A healthy pool shows `ONLINE` for all components. `DEGRADED` means a drive has failed but the pool is still running. `FAULTED` means the pool is down.
