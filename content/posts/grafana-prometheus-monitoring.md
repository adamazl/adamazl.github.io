---
title: "Homelab Monitoring with Prometheus and Grafana"
date: 2026-03-07
draft: false
description: "Build a complete monitoring stack for your homelab using Prometheus for metrics collection and Grafana for dashboards."
tags: ["grafana", "prometheus", "monitoring", "observability", "docker", "homelab"]
categories: ["Infrastructure"]
showToc: true
---

## Why Monitor Your Homelab?

Without monitoring, you find out about problems when something stops working. With it, you see a disk filling up days before it causes an outage, catch a VM chewing CPU in the middle of the night, or notice your UPS battery health declining. It turns reactive firefighting into proactive maintenance.

The Prometheus + Grafana stack is the industry standard for this and runs comfortably on modest hardware. My monitoring stack runs in Docker on a dedicated LXC container and uses less than 1 GB RAM.

## The Stack

- **Prometheus** — time-series database that scrapes metrics endpoints on a schedule
- **node_exporter** — exposes Linux system metrics (CPU, RAM, disk, network) as a Prometheus endpoint
- **Grafana** — visualization layer; connects to Prometheus and renders dashboards
- **Alertmanager** (optional) — sends alerts when thresholds are breached

## Docker Compose Setup

Create `/opt/monitoring/docker-compose.yml`:

```yaml
version: '3'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=90d'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=your-password-here
      - GF_USERS_ALLOW_SIGN_UP=false

volumes:
  prometheus_data:
  grafana_data:
```

## Prometheus Configuration

Create `/opt/monitoring/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets:
          - 'pve-01:9100'
          - 'pve-02:9100'
          - 'nas-01:9100'
          - 'pihole:9100'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+):.*'
        replacement: '$1'
```

Add every machine you want to monitor as a target.

## Installing node_exporter

On each Linux machine you want to monitor:

```bash
# Download latest release
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar xf node_exporter-*.tar.gz
cp node_exporter-*/node_exporter /usr/local/bin/
chmod +x /usr/local/bin/node_exporter
```

Create a systemd service at `/etc/systemd/system/node_exporter.service`:

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=nobody
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now node_exporter
```

Verify it's working: `curl http://localhost:9100/metrics | head -20`

## Proxmox Metrics

For Proxmox VE itself, use [pve_exporter](https://github.com/prometheus-pve/prometheus-pve-exporter):

```yaml
# Add to docker-compose.yml
  pve-exporter:
    image: prompve/prometheus-pve-exporter:latest
    container_name: pve-exporter
    restart: unless-stopped
    ports:
      - "9221:9221"
    volumes:
      - ./pve.yml:/etc/pve.yml:ro
```

Create `/opt/monitoring/pve.yml`:

```yaml
default:
  user: prometheus@pve
  password: your-password
  verify_ssl: false
```

Create a `prometheus@pve` user in Proxmox with read-only access to the datacenter.

Add to `prometheus.yml`:

```yaml
  - job_name: 'proxmox'
    metrics_path: /pve
    params:
      module: [default]
    static_configs:
      - targets: ['192.168.1.10']
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: 'pve-exporter:9221'
```

## Start the Stack

```bash
cd /opt/monitoring
docker compose up -d
```

Access Prometheus at `http://monitoring-ip:9090` and Grafana at `http://monitoring-ip:3000`.

## Grafana Setup

1. Log in with admin / your-password-here
2. **Connections -> Add new connection -> Prometheus**
3. URL: `http://prometheus:9090` (Docker internal network)
4. Click **Save & test**

## Importing Dashboards

Grafana has a dashboard library at grafana.com/dashboards. Import by ID:

| Dashboard | ID |
|-----------|-----|
| Node Exporter Full | 1860 |
| Proxmox via pve_exporter | 10347 |
| Docker containers | 893 |
| Pi-hole | 10176 |

Go to **Dashboards -> Import**, paste the ID, select your Prometheus data source.

The Node Exporter Full dashboard (ID 1860) is particularly good — it covers CPU usage, memory, disk I/O, network throughput, and filesystem usage all in one view.

## Setting Up Alerts

In Grafana, go to **Alerting -> Alert rules -> New alert rule**:

Example — disk space alert:

```promql
(node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
```

This fires when root filesystem is less than 15% free. Set it to send an email or push notification via a contact point.

## Useful PromQL Queries

```promql
# CPU usage percentage
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100

# Disk I/O (reads per second)
rate(node_disk_reads_completed_total[5m])

# Network throughput
rate(node_network_receive_bytes_total[5m])
```

These are great as Grafana panel queries when building custom dashboards.
