---
title: "Installing Proxmox VE"
date: 2026-03-09
draft: false
description: "A step-by-step walkthrough of how I installed Proxmox VE on bare metal for my homelab compute nodes."
tags: ["proxmox", "virtualisation", "homelab", "linux"]
categories: ["Infrastructure"]
series: ["Home Network Build"]
showToc: true
---

## What is Proxmox VE?

Proxmox Virtual Environment (VE) is a free, open-source hypervisor built on Debian. It supports both
KVM-based virtual machines and LXC containers, and comes with a decent web UI out of the box — no
need to pay for a VMware licence.

## Hardware

For this build I'm running Proxmox on two nodes:

| Node | CPU | RAM | Storage |
|------|-----|-----|---------|
| pve-01 | Intel Core i5-12400 | 32 GB DDR4 | 500 GB NVMe (OS) + 2 TB SSD (VMs) |
| pve-02 | Intel Core i5-10400 | 16 GB DDR4 | 256 GB NVMe (OS) + 1 TB SSD (VMs) |

## Downloading the ISO

Head to the [Proxmox downloads page](https://www.proxmox.com/en/downloads) and grab the latest
**Proxmox VE ISO Installer**. At the time of writing that was 8.x.

Flash it to a USB drive with something like [Balena Etcher](https://etcher.balena.io/) or `dd`:

```bash
dd if=proxmox-ve_*.iso of=/dev/sdX bs=1M status=progress
```

Replace `/dev/sdX` with your actual USB device — double-check with `lsblk` before running this.

## Installation

1. Boot from the USB. You may need to enter the BIOS (`Del` / `F2` / `F12` depending on your board)
   and either change the boot order or use the one-time boot menu.

2. Select **Install Proxmox VE (Graphical)**.

3. Accept the EULA.

4. **Target disk** — select your intended OS drive. I used the NVMe. Leave the filesystem as `ext4`
   unless you have a reason to use ZFS (ZFS is great but needs RAM).

5. **Location and timezone** — set your country and timezone.

6. **Password and email** — set a strong root password. The email is used for system notifications.

7. **Network configuration** — this is important:
   - Set a static IP for the management interface (e.g. `192.168.1.10/24`)
   - Set the gateway to your router's IP
   - Set a DNS server (I use my Pi-hole IP, falling back to `1.1.1.1`)
   - Give the node a hostname like `pve-01.local`

8. Review the summary and click **Install**. The process takes a few minutes.

9. Remove the USB when prompted and let the node reboot.

## First Login

Once it's back up, open a browser and navigate to:

```
https://192.168.1.10:8006
```

You'll get a certificate warning — that's expected for a self-signed cert. Accept it and log in with
`root` and the password you set during install.

## Post-Install Steps

### Disable the Enterprise Repo (no subscription)

By default Proxmox points at the enterprise apt repository, which requires a paid subscription. If
you don't have one, switch to the no-subscription repo to get updates.

In the web UI: **Node > Shell**, then:

```bash
# Disable enterprise repo
echo "# disabled" > /etc/apt/sources.list.d/pve-enterprise.list

# Add no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

apt update && apt dist-upgrade -y
```

### Remove the Subscription Nag

The web UI shows a "No valid subscription" popup on every login. You can patch it out:

```bash
sed -i.bak "s/data.status !== 'Active'/false/g" \
  /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js

systemctl restart pveproxy
```

### Enable IOMMU (for PCIe Passthrough)

If you plan to pass through a GPU or other PCIe device, enable IOMMU in the bootloader config.

For systems using GRUB:

```bash
# /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

update-grub
reboot
```

For AMD CPUs replace `intel_iommu=on` with `amd_iommu=on`.

Verify after reboot:

```bash
dmesg | grep -e DMAR -e IOMMU
```

### Set Up Storage

In the web UI, go to **Datacenter > Storage** to add your VM storage disk. I added the 2 TB SSD as
a `Directory` type datastore, pointed at a partition mounted at `/mnt/data`.

## What's Next

With both nodes up, the next step is connecting them into a cluster and setting up
[Proxmox Backup Server](https://www.proxmox.com/en/proxmox-backup-server) on dedicated hardware for
automated VM backups.
