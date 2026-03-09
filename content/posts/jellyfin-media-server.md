---
title: "Self-Hosted Media Streaming with Jellyfin"
date: 2026-03-02
draft: false
description: "Set up Jellyfin as a self-hosted media server to stream your film and music library to any device."
tags: ["jellyfin", "media", "streaming", "docker", "homelab"]
categories: ["Infrastructure"]
showToc: true
---

## Why Jellyfin?

Jellyfin is a free, open-source media server — the community fork of Emby after Emby went partially closed-source. It lets you stream your personal media library (films, TV shows, music, photos) to any device through a browser or app.

Compared to Plex, Jellyfin is fully free with no premium tier. Transcoding, sync, and apps are all free. There's no phoning home to Plex servers — all metadata and authentication is local.

## Hardware Requirements

Jellyfin can serve media in two ways:

- **Direct play:** the client plays the file as-is. Zero server CPU required. Works when the client can handle the codec.
- **Transcoding:** the server converts the video on the fly. Requires CPU (or GPU) power.

For direct play only, any old PC handles it. For transcoding (needed for older TVs, browsers, or multiple simultaneous streams), you want either a fast CPU or hardware transcoding via a GPU or Intel QuickSync.

Intel QuickSync (available on most Intel CPUs since Ivy Bridge) handles hardware transcoding efficiently. An Intel N100 or i5-12400 can transcode 4–8 simultaneous 1080p streams.

## Installation with Docker

Create `/opt/jellyfin/docker-compose.yml`:

```yaml
version: '3'
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    network_mode: host  # required for DLNA discovery
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /mnt/nas/media:/media:ro  # your media library, read-only
    environment:
      - JELLYFIN_PublishedServerUrl=https://jellyfin.yourdomain.com
    devices:
      - /dev/dri:/dev/dri  # Intel QuickSync hardware transcoding
```

The `/dev/dri` device passthrough enables hardware transcoding. This works in Docker on bare metal and in a privileged LXC container.

Start it:

```bash
docker compose up -d
```

Access at `http://server-ip:8096` for first-time setup.

## First-Time Setup

The setup wizard asks for:

1. **Admin username/password** — set something strong
2. **Media libraries** — add your libraries:
   - **Movies** — point to `/media/movies`
   - **TV Shows** — point to `/media/tv`
   - **Music** — point to `/media/music`

Jellyfin scans the library and fetches metadata (posters, descriptions, ratings) from TheMovieDB and TheTVDB automatically.

## Library Structure

Jellyfin expects a specific folder structure for best metadata matching:

```
/media/
  movies/
    The Matrix (1999)/
      The Matrix (1999).mkv
    Interstellar (2014)/
      Interstellar (2014).mkv
  tv/
    Breaking Bad/
      Season 01/
        Breaking Bad - S01E01 - Pilot.mkv
```

The year in parentheses helps with disambiguation. For TV, the `S01E01` naming convention is important for episode matching.

## Hardware Transcoding Setup

In Jellyfin: **Dashboard -> Playback -> Transcoding**

- **Hardware acceleration:** Intel QuickSync Video (QSV)
- Enable HEVC encoding if your iGPU supports it (most 6th gen+ Intel do)
- Enable **Tone mapping** for HDR-to-SDR conversion

Verify it's working by starting a transcode and checking **Dashboard -> Active Sessions** — you should see `(hw)` next to the codec.

## Clients

Jellyfin has apps for:

- **Web browser** — works well, no install needed
- **Android / iOS** — official Jellyfin app
- **Android TV / Fire TV** — excellent, supports direct play of most formats
- **Apple TV** — Infuse (paid) or Swiftfin (free) work well
- **Kodi** — Jellyfin Kodi addon for a full HTPC setup
- **Samsung/LG Smart TV** — browser or native apps via app stores

## Remote Access

For remote access outside your home network, put Jellyfin behind Nginx Proxy Manager with a Let's Encrypt certificate. Set `JELLYFIN_PublishedServerUrl` to your external URL.

Under **Dashboard -> Networking**, also set the external URL so Jellyfin uses it in links and API responses.

## Media Management Companions

Jellyfin handles serving, but for managing your library:

- **Sonarr** — automatically downloads TV show episodes
- **Radarr** — automatically downloads movies
- **Prowlarr** — indexes Usenet and torrent indexers for Sonarr/Radarr
- **Bazarr** — automatic subtitle download

These form the "arr stack" and integrate with each other. With it set up, you add a show to Sonarr and new episodes download automatically, appear in your NAS, and show up in Jellyfin.

## Plugins

Jellyfin has a plugin catalog. Useful ones:

- **Intro Skipper** — detects and skips TV show intros
- **Merge Versions** — merges multiple versions of the same film into one library item
- **Playback Reporting** — tracks what you've watched with statistics

Install from **Dashboard -> Plugins -> Catalog**.
