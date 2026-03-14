---
title: "Getting Started with Home Assistant"
date: 2026-03-01
draft: false
description: "Set up Home Assistant on Proxmox to automate your smart home devices with full local control."
tags: ["homeassistant", "smarthome", "automation", "proxmox", "homelab"]
categories: ["Infrastructure"]
showToc: true
---

## Why Home Assistant?

Most smart home ecosystems (Google Home, Amazon Alexa, Apple HomeKit) are cloud-dependent. Your lights, locks, and sensors phone home to a server. If the company shuts down, changes their API, or has an outage, your devices stop working.

Home Assistant is a local, open-source smart home platform. It runs on your network, integrates with over 3,000 services and devices, and keeps all automations local. Your automations run even when the internet is down.

## Running Home Assistant on Proxmox

The recommended way to run Home Assistant is as the **Home Assistant OS (HAOS)** — a complete appliance image. It manages updates, add-ons, and backups through a clean UI.

### Import the HAOS VM Image

On your Proxmox node:

```bash
# Download the Proxmox-specific image
wget https://github.com/home-assistant/operating-system/releases/download/13.2/haos_ova-13.2.qcow2.xz
xz -d haos_ova-13.2.qcow2.xz
```

Create a new VM in Proxmox (via UI or CLI):

- **OS:** Do not use any media
- **System:** UEFI, SCSI controller, machine q35
- **Disk:** Delete the default disk — you'll import the image
- **CPU:** 2 cores
- **Memory:** 4 GB (2 GB minimum, 4 GB if running many integrations)
- **Network:** VirtIO

Import the disk:

```bash
qm importdisk 100 haos_ova-13.2.qcow2 local-lvm
```

In the VM's **Hardware** tab, attach the imported disk (Unused Disk 0) as the boot disk. Set boot order to this disk.

Start the VM and navigate to `http://homeassistant.local:8123` or the VM's IP on port 8123.

## Initial Setup

The first boot takes a few minutes to provision. The onboarding wizard asks for:

1. Your name and location (used for sun-based automations, local time)
2. Whether to share analytics (optional)
3. Home Assistant creates a local account — this is not cloud-based

## Adding Integrations

Go to **Settings -> Devices & Services -> Add Integration**. Home Assistant auto-discovers many devices on your network (Philips Hue, Sonos, Chromecasts, etc.).

Popular integrations:

- **Zigbee Home Automation (ZHA)** — native Zigbee coordinator support
- **Z-Wave JS** — Z-Wave devices
- **MQTT** — for DIY sensors and many IoT devices
- **ESPHome** — custom ESP8266/ESP32 sensors
- **Philips Hue** — via local API (no cloud)
- **Sonos** — local control
- **Unifi** — presence detection via network
- **OPNsense** — network stats
- **HACS (Home Assistant Community Store)** — third-party integrations

## USB Dongle Passthrough in Proxmox

For Zigbee or Z-Wave, you need a USB coordinator dongle passed through to the HA VM.

In Proxmox, go to the VM's **Hardware -> Add -> USB Device**:

- **Use USB Vendor/Device ID** — select your dongle from the list

Popular dongles:
- **Sonoff Zigbee 3.0 USB Dongle Plus** (~$20) — excellent Zigbee support
- **HUSBZB-1** — dual Zigbee/Z-Wave, useful if you have both protocols

In Home Assistant, add the **Zigbee Home Automation** integration and point it at your USB device (typically `/dev/ttyUSB0` or `/dev/serial/by-id/...`).

## Automations

Automations consist of:

- **Trigger** — what starts the automation (time, state change, event)
- **Condition** (optional) — when it should run (e.g., only if someone is home)
- **Action** — what to do (turn on lights, send notification, etc.)

Example automation: turn on lights at sunset

```yaml
alias: Evening lights
trigger:
  - platform: sun
    event: sunset
    offset: "-00:30:00"
condition:
  - condition: state
    entity_id: person.adam
    state: home
action:
  - service: light.turn_on
    target:
      area_id: living_room
    data:
      brightness_pct: 60
      color_temp: 3000
```

## Add-ons

Home Assistant OS supports add-ons — containerised apps that run alongside HA and integrate tightly:

- **Mosquitto Broker** — local MQTT broker for IoT devices
- **ESPHome** — manage custom ESP32 sensors
- **Node-RED** — visual automation builder (more powerful than built-in automations)
- **Zigbee2MQTT** — alternative Zigbee coordinator (more device support than ZHA)
- **Samba** — share the HA config directory over SMB for easy editing
- **VS Code Server** — edit config files in a browser-based VS Code

Install from **Settings -> Add-ons -> Add-on Store**.

## Dashboards

Home Assistant's dashboard (Lovelace) is highly customisable. Go to **Overview** and click the pencil icon to edit.

The **HACS** store has hundreds of custom cards:

- **Mini Graph Card** — compact line graphs for sensors
- **Mushroom Cards** — clean, modern card designs
- **ApexCharts Card** — advanced graphing

## Presence Detection

Home Assistant can determine who's home via:

- **Mobile app** (iPhone/Android) — GPS + local Wi-Fi detection
- **Network scanner** — detect devices by MAC address on the network
- **Bluetooth** — detect BLE devices
- **UniFi integration** — read connected clients from UniFi controller

Presence detection powers automations like "turn off everything when everyone leaves" and "unlock the door when I arrive home".

## Backups

Home Assistant has built-in backup under **Settings -> System -> Backups**. Backups are `.tar` files containing your config, automations, and database.

Store backups offsite: configure the **Google Drive Backup** add-on or manually sync to your NAS. You can also restore entirely from a backup onto new hardware — HA backups are self-contained.
