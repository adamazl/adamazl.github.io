---
title: "Reverse Proxy and SSL with Nginx Proxy Manager"
date: 2026-02-10
draft: false
description: "Using Nginx Proxy Manager to front your self-hosted services with real SSL certificates, custom domain names, and access control — without touching raw Nginx config."
tags: ["nginx", "reverse-proxy", "ssl", "docker", "homelab", "networking"]
categories: ["Networking"]
series: ["Home Network Build"]
showToc: true
---

## The Problem Nginx Proxy Manager Solves

As your homelab grows, you accumulate services running on various IPs and ports: Proxmox on `:8006`, Jellyfin on `:8096`, Nextcloud on `:443`, Home Assistant on `:8123`. Remembering port numbers is tedious, but the bigger issue is HTTPS — browsers complain about self-signed certificates, and accessing services over plain HTTP on your LAN is a security risk.

Nginx Proxy Manager (NPM) solves both problems. It's a Docker container with a web UI that lets you:

- Route traffic by hostname (`jellyfin.home.lan`) to internal services
- Automatically issue and renew Let's Encrypt SSL certificates
- Add authentication to services that don't have their own login
- Without writing a single line of Nginx config

## DNS Setup First

NPM works by routing based on hostname. Before you set it up, decide on your domain strategy.

**Option 1 — Local domain with Pi-hole/local DNS.** Use a fake TLD like `.lan` or `.home.arpa`. Works for internal access only. SSL via Let's Encrypt is not possible (Let's Encrypt won't issue certs for `.lan`), so you'd need self-signed certs.

**Option 2 — Real domain with split DNS.** Own a real domain (`yourdomain.com`) and create a subdomain (`*.home.yourdomain.com`) that points to your NPM IP in your local DNS server (Pi-hole). The same subdomain points to nothing on the public internet. Let's Encrypt can issue real certs via DNS challenge, so you get trusted SSL internally.

**Option 3 — Public subdomain.** Point `*.yourdomain.com` to your public IP (or use Cloudflare proxy). Requires port 80/443 forwarded to NPM. Works for external access too.

Option 2 is what I use and what this guide covers.

## Installing Nginx Proxy Manager

NPM runs as a Docker container. I deploy it in an LXC container on Proxmox (with Docker installed), though a dedicated VM works too.

Create `docker-compose.yml`:

```yaml
services:
  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"   # Admin UI
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

Start it:

```bash
docker compose up -d
```

Access the admin UI at `http://<host-ip>:81`. Default credentials:

```
Email:    admin@example.com
Password: changeme
```

Change both immediately after first login.

## Adding Your First Proxy Host

Let's add a proxy for Jellyfin running at `192.168.1.50:8096`.

1. **Hosts > Proxy Hosts > Add Proxy Host**
2. **Domain Names**: `jellyfin.home.yourdomain.com`
3. **Scheme**: `http` (Jellyfin's internal traffic is HTTP)
4. **Forward Hostname/IP**: `192.168.1.50`
5. **Forward Port**: `8096`
6. Enable **Websockets Support** (Jellyfin needs this)
7. On the **SSL** tab: Select **Request a new SSL Certificate**

Before the SSL cert can be issued, there's a DNS configuration step.

## Issuing SSL Certificates with DNS Challenge

For internal services (not publicly accessible), Let's Encrypt can't use the HTTP challenge. Use the DNS challenge instead. This requires your domain to be managed by a supported DNS provider — Cloudflare is the easiest.

### Setting Up Cloudflare DNS

1. Transfer your domain to Cloudflare (free plan is fine)
2. In Cloudflare, go to **My Profile > API Tokens > Create Token**
3. Use the **Edit zone DNS** template, scoped to your specific domain
4. Copy the token

In NPM, when requesting a cert, select **DNS Challenge** and choose **Cloudflare**. Paste your API token. NPM will automatically create the `_acme-challenge` TXT record, verify it, and issue the cert.

The cert is valid for 90 days. NPM renews it automatically. You never have to think about it.

### Wildcard Certificates

Instead of issuing individual certs per service, issue a wildcard: `*.home.yourdomain.com`. This cert covers all subdomains.

In NPM: **SSL Certificates > Add SSL Certificate > Let's Encrypt**

- Domain names: `*.home.yourdomain.com` and `home.yourdomain.com`
- DNS Challenge: Cloudflare
- Your API token

One cert, every subdomain. When you add new proxy hosts, select this cert instead of requesting a new one.

## Local DNS Records for Each Service

In Pi-hole (or your local DNS), create an A record for each subdomain pointing to the NPM host IP:

```
jellyfin.home.yourdomain.com  -> 192.168.1.5  (NPM's IP)
nextcloud.home.yourdomain.com -> 192.168.1.5
proxmox.home.yourdomain.com   -> 192.168.1.5
```

All traffic hits NPM, which routes to the right backend based on hostname. The actual services can stay on their existing IPs and ports — nothing changes on their end.

## Access Lists: Adding Authentication

Some services (like Prometheus metrics endpoints) don't have their own login. NPM can add HTTP Basic Auth in front of them.

**Access Lists > Add Access List**:
- Add username/password pairs
- Optionally add IP whitelist entries (e.g., only allow access from your LAN subnet)

Assign an access list to a proxy host under the **Access > Access List** dropdown.

This is useful for:
- Admin UIs you don't want accidentally exposed
- Grafana (add it even if Grafana has its own auth — defense in depth)
- Services without authentication, like node-exporter metrics

## Proxmox Behind NPM

Proxmox's web UI uses self-signed certs and runs on port 8006. A few extra steps are needed:

1. In NPM, set the scheme to `https` (Proxmox uses HTTPS internally)
2. Enable **Verify SSL** OFF — NPM needs to trust Proxmox's self-signed cert
3. Enable **Websockets Support** — the Proxmox console uses websockets

```
Scheme: https
Forward Host: 192.168.1.10
Forward Port: 8006
Websockets: enabled
SSL verification: disabled
```

Result: `https://proxmox.home.yourdomain.com` with a valid cert, routing to your Proxmox node.

## Custom 404 and Error Pages

Under **Settings > Default Site**, configure what happens for requests that don't match any proxy host. Options are a 404 page or a redirect. I redirect to NPM's own admin page, but a custom 404 is cleaner if NPM is exposed externally.

## Keeping NPM Updated

NPM is actively maintained. To update:

```bash
docker compose pull
docker compose up -d
```

Configuration and certificates are stored in the `./data` and `./letsencrypt` volumes, so they survive the update.

## Alternatives Worth Knowing

- **Traefik** — more powerful, config-as-code, native Docker/Kubernetes integration. Steeper learning curve. Better choice if you want infrastructure-as-code.
- **Caddy** — automatic HTTPS with a clean config syntax. No UI, but the config file is simple.
- **Authelia/Authentik** — NPM handles basic auth, but if you want SSO with MFA across all your services, add Authentik in front of NPM.

For a homelab without dozens of services, NPM's UI-driven approach is hard to beat for speed of setup.
