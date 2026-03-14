# The Home Lab

A personal blog about homelabbing, self-hosting, and home networking — built with [Hugo](https://gohugo.io/) and deployed to GitHub Pages.

**Live site: [adamazl.github.io/homelab](https://adamazl.github.io/homelab/)**

## Tech Stack

- **Framework:** Hugo (Extended v0.157.0)
- **Theme:** Custom (PaperMod-based)
- **Styling:** Tailwind CSS
- **Deployment:** GitHub Actions → GitHub Pages

## Features

- Dark/light mode toggle
- Full-text search
- Code blocks with copy buttons
- Callout/admonition components
- Table of contents on posts
- Shadcn-inspired UI

## Topics Covered

- Proxmox VE & virtualization
- TrueNAS & ZFS storage
- Docker & LXC containers
- Network setup (OPNsense, VLANs, WireGuard, Tailscale)
- Self-hosted services (Jellyfin, Home Assistant, Pi-hole, Nginx Proxy Manager, Immich)
- Monitoring (Grafana + Prometheus)
- Backup strategies (Proxmox Backup Server)

## Local Development

**Prerequisites:** Hugo Extended, Node.js 20+

```bash
# Clone the repo
git clone --recurse-submodules https://github.com/adamazl/homelab.git
cd homelab

# Install dependencies
npm install

# Start dev server
hugo server -D
```

The site will be available at `http://localhost:1313/homelab/`.

## Deployment

Pushing to `main` automatically triggers the GitHub Actions workflow which:
1. Builds Tailwind CSS
2. Builds the Hugo site
3. Deploys to GitHub Pages

## License

Content is my own. Feel free to use code snippets with attribution.
