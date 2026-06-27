# Ironhide Homelab

Documenting my homelab build, from thrifted PC case to a multi-VM wireless workhorse.

## Overview

This repo tracks the build, configuration, and ongoing development of my home Proxmox server, named **Ironhide**, along with the rest of the devices "crew" that connects to it.

## The Crew

| Name | Role | Specs |
|------|------|-------|
| **Ironhide** | Homelab server | Ryzen 5 3600, 32GB DDR4 3200MHz, Fractal Torrent case |
| **Magnus** | Daily driver desktop | Ryzen 7 1700X, RTX 2060 Super, 32GB RAM |
| **Soundwave** | Portable / remote access | Lenovo ThinkPad T14 Gen 4 |

## Ironhide Specs

- **CPU:** AMD Ryzen 5 3600
- **Motherboard:** MSI B450 Gaming Plus
- **RAM:** 32GB DDR4 3200MHz
- **OS Drive:** WD Blue SN570 250GB NVMe
- **VM Storage:** Samsung 860 EVO 1TB SSD
- **NAS Storage:** Seagate IronWolf 2TB HDD
- **PSU:** Corsair CX550
- **Case:** Fractal Torrent (thrifted)

## Current Status

- ✅ Proxmox VE installed and configured
- ✅ Tailscale mesh network across all devices
- ✅ Ubuntu Server VM — running, SSH accessible
- ✅ Kali Linux VM — running, SSH + VNC desktop
- ✅ TrueNAS VM — running, storage pool configured
- ⬜ BIOS update pending
- ⬜ Additional networking hardware setup

## What It Hosts

Ironhide doesn't just run VMs for the sake of it — it actually serves things. Right now:

- **[seaksolutions.dev](https://seaksolutions.dev)** — my live portfolio and business site, hand-coded HTML/CSS/JS, served by Nginx on the Ubuntu Server VM, exposed to the public internet through a Cloudflare Tunnel (no open ports on my router). Hardened with UFW, fail2ban, and automatic security updates.
  Source: **[github.com/SeanKoal/seak-solutions-website](https://github.com/SeanKoal/seak-solutions-website)**

## Documentation

More detailed write-ups coming soon in the `/docs` folder, including hardware build notes, networking setup, and a troubleshooting log of real problems solved along the way.

## About Me

Computer Programming and Information Science student, pursuing CompTIA A+, Network+, and Security+. Building this homelab to learn hands on virtualization, networking, and security concepts.
