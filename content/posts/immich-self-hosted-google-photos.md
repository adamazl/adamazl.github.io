---
title: "Immich: Your Self-Hosted Google Photos Replacement"
date: 2026-03-14
tags: ["immich", "photos", "self-hosted", "docker", "storage", "homelab"]
categories: ["Infrastructure"]
description: "Set up Immich on your homelab to get Google Photos-style automatic backup, facial recognition, and AI-powered search — with full control over your data."
draft: false
showToc: true
---

## Why You Need This

Let me paint you a picture. You've got 80,000 photos scattered across your phone, your partner's phone, and three old laptops. They're on Google Photos — until Google changes their storage policy again, or you get locked out of your account, or you just get tired of feeding a trillion-dollar company your entire family's memories.

Or maybe you're already off Google and your photos are just... sitting in a folder on your NAS. Perfectly preserved, completely unsearchable, impossible to share.

Immich fixes this. It's a self-hosted photo and video backup platform that feels eerily like Google Photos — automatic mobile backup, a slick web UI, face grouping, map view, "on this day" memories, and machine-learning-powered search. Type "beach sunset 2023" and it finds it. It's that good.

This is consistently one of the most upvoted projects on r/selfhosted for good reason. NetworkChuck called it "the most impressive self-hosted app I've ever used" and honestly? Hard to argue with that.

> **Heads up:** Immich is under very active development. The team ships updates constantly, which is great for features but means config formats occasionally change. Always check the [official docs](https://immich.app/docs) alongside this guide.

---

## What You'll Need

- A host with **Docker and Docker Compose** (an LXC container on Proxmox works great — see my [Docker vs LXC post](/posts/docker-vs-lxc-proxmox/))
- At least **4 GB RAM** (8 GB recommended if you want machine learning features)
- **Plenty of storage** — photos and videos add up fast. Point this at your NAS share or ZFS dataset (see my [ZFS post](/posts/zfs-storage-pools/))
- A **reverse proxy** if you want HTTPS and a clean domain name (my [Nginx Proxy Manager guide](/posts/nginx-proxy-manager-ssl/) covers this)
- The **Immich mobile app** on iOS or Android (free, on the App Store and Play Store)

Minimum viable CPU: anything modern works. Machine learning (face detection, CLIP search) runs on the CPU by default, but Immich supports GPU acceleration via CUDA or OpenCL if you have an Nvidia or Intel GPU — big speedup for large libraries.

---

## Architecture Overview

Immich is a multi-container application. Here's what actually runs when you `docker compose up`:

| Container | What It Does |
|---|---|
| `immich-server` | Main API and web UI |
| `immich-microservices` | Background jobs: thumbnail generation, ML inference, metadata extraction |
| `immich-machine-learning` | The AI model server (face detection, semantic search) |
| `redis` | Job queue and caching |
| `postgres` | Database (with the `pgvecto-rs` extension for vector search) |

It sounds like a lot, but Docker Compose manages it all as a single unit. The official `docker-compose.yml` is well-tuned and just works.

---

## Step 1: Prepare Your Storage

Before touching Docker, decide where your photos will actually live. This is the most important decision you'll make — photos are irreplaceable.

I store mine on a dedicated ZFS dataset on my NAS, mounted into the Immich host via NFS:

```bash
# On TrueNAS / ZFS host — create a dedicated dataset
zfs create tank/immich
zfs create tank/immich/library    # actual photos
zfs create tank/immich/thumbs     # thumbnails (can be regenerated)
zfs create tank/immich/profile    # profile pictures
zfs create tank/immich/upload     # staging area

# Set a quota if you want to cap it
zfs set quota=2T tank/immich/library
```

If you're not on ZFS yet, at minimum make sure Immich's library folder is on whatever drive your important data lives on, and that you're including it in your backups.

> **Gotcha:** Don't point Immich's library at the same location as your existing manually-organised photo folders. Immich manages its own internal folder structure. Use the "External Libraries" feature (covered below) for existing organised collections.

---

## Step 2: Create the Docker Compose Stack

Create a working directory and grab the official files:

```bash
mkdir -p /opt/immich && cd /opt/immich

# Grab the official compose file and env template
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

Now edit `.env` — this is where you set the important bits:

```bash
# .env

# Database password — change this to something strong
DB_PASSWORD=change_me_to_something_strong

# Immich version — pin this! Don't use "latest" in production.
# Check https://github.com/immich-app/immich/releases for current version
IMMICH_VERSION=v1.130.0

# Where photos are stored on the host
UPLOAD_LOCATION=/mnt/tank/immich/library

# Where the database stores its files
DB_DATA_LOCATION=/opt/immich/postgres
```

> **Pin your version.** I can't stress this enough. Using `latest` means an auto-update could break your stack at 2 AM. Pin to a specific version, read the release notes, then upgrade deliberately.

Now look at the `docker-compose.yml`. The default file is solid, but there are a few tweaks worth making:

```yaml
services:
  immich-server:
    deploy:
      resources:
        limits:
          memory: 2g

  immich-machine-learning:
    volumes:
      - model-cache:/cache

volumes:
  model-cache:
```

---

## Step 3: Bring It Up

```bash
cd /opt/immich
docker compose up -d
```

Watch the logs while everything initialises:

```bash
docker compose logs -f immich-server
```

First boot takes a minute or two. When you see:

```
immich-server | [Nest] LOG [NestApplication] Nest application successfully started
```

You're live. Hit `http://your-server-ip:2283` in a browser.

---

## Step 4: Initial Setup

**Create your admin account.** The first user you create becomes the admin.

**Walk through the onboarding wizard.** It'll ask about your storage and show you the mobile app QR code.

**Set your server URL.** In the admin panel, go to **Administration > Settings > General**. Set the "External Domain" to the URL your mobile app will use to connect.

---

## Step 5: Mobile App Setup

Install the Immich app (iOS or Android). Tap **Login**, enter your server URL, and log in.

Go to **Settings > Background Backup** and enable it. The app uploads your camera roll in the background, WiFi-only by default, and resumes from where it left off.

For family members, create additional accounts in **Administration > Users**. Each person gets their own library with optional album sharing.

---

## Step 6: Add a Reverse Proxy and HTTPS

If you're using Nginx Proxy Manager (covered in my [NPM guide](/posts/nginx-proxy-manager-ssl/)), add a proxy host:

- **Domain Names:** `photos.yourdomain.com`
- **Scheme:** `http`
- **Forward Hostname / IP:** your host IP
- **Forward Port:** `2283`
- **SSL:** Request a Let's Encrypt cert, enable "Force SSL"

> **Tip:** If you're using Tailscale (see my [Tailscale guide](/posts/tailscale-remote-access/)), you can skip the public domain entirely and use your Tailscale IP. The mobile app works fine over Tailscale.

---

## Step 7: The Good Stuff — Features Worth Knowing

### Semantic Search (This Is the Killer Feature)

Immich uses CLIP to understand what's *in* your photos without you tagging anything. Try searching:

- `"red dress at a wedding"`
- `"dog at the beach"`
- `"mountain view at sunset"`

It searches by meaning, not filenames. This is the feature that makes people drop Google Photos immediately.

### Face Recognition

Immich detects and clusters faces automatically. Go to **Explore > People** to see grouped faces, name them, then search `"photos with [name]"`. Runs entirely locally — your family's faces never leave your server.

### Map View

If your photos have GPS metadata (most phone photos do), the map view shows a globe of everywhere you've taken photos. Genuinely beautiful.

### External Libraries

Have an existing structured photo archive you don't want Immich to reorganise? Use External Libraries:

1. Go to **Administration > External Libraries**
2. Add a bind mount in `docker-compose.yml`:

```yaml
services:
  immich-server:
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /mnt/tank/photos/archive:/mnt/archive:ro
```

3. Point the External Library at `/mnt/archive` — Immich indexes files in place without moving them.

---

## Backup Strategy

**Immich is not a backup — it's a primary store.** The 3-2-1 rule applies:

1. Primary: ZFS dataset on TrueNAS
2. Local backup: ZFS send/receive to a second pool
3. Offsite: Rclone sync to Backblaze B2 (~$6/TB/month)

The Postgres database also needs backing up — it stores all your albums, face names, and metadata:

```bash
# Dump the database (run on a schedule)
docker exec -t immich_postgres pg_dumpall -c -U postgres > /backup/immich-db-$(date +%Y%m%d).sql

# Keep only the last 30 days
find /backup -name "immich-db-*.sql" -mtime +30 -delete
```

---

## Upgrading Immich

```bash
cd /opt/immich

# Bump IMMICH_VERSION in .env, then:
docker compose pull
docker compose up -d
docker compose logs -f immich-server
```

> **Always read the release notes before upgrading.** Immich occasionally has breaking changes. The team documents them clearly — don't skip them.

---

## Performance Tuning

**Machine learning is slow on first run** for large libraries. It runs in the background — don't panic if face search isn't instant.

**GPU acceleration** for Nvidia GPUs:

```yaml
  immich-machine-learning:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

```bash
# In .env
MACHINE_LEARNING_DEVICE=cuda
```

Even a modest GPU turns hours of ML processing into minutes.

---

## Common Gotchas

**Photos not uploading from mobile** — check that your server URL is reachable outside your home network.

**ML container keeps crashing** — almost always a RAM issue. Needs at least 2 GB. You can disable ML entirely in admin settings if you're RAM-constrained.

**Accidentally deleted photos** — check the Trash. Immich soft-deletes with a 30-day retention by default.

**Thumbnails look blurry** — they're generated in the background after import. Watch job progress in **Administration > Jobs**.

---

## What's Next

- **Share albums with family** — invite by email, no account required for public links
- **Webhooks + Home Assistant** — trigger automations when new photos arrive
- **Immich CLI** — bulk import existing archives from the command line
- **Hardware transcoding** — for video-heavy libraries, enable in FFMPEG settings

The Immich community on GitHub Discussions and r/selfhosted is incredibly active. Whatever weird thing you're trying to do, someone's probably done it.

---

Your photos are your memories. They deserve to live somewhere you control, on hardware you own, with software you can inspect. Immich makes that genuinely pleasant — and once your family's phones are backing up automatically, they'll never know the difference from Google Photos. Except the photos will still be there in 10 years, exactly where you left them.

Now go reclaim your data.
