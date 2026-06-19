# Ubuntu Server VM

## Purpose

A general-purpose Linux server, intended as a flexible foundation for learning command-line administration and eventually hosting real services — a personal Git server, web app, or file sharing setup.

## Specs

- **OS:** Ubuntu Server 26.04 LTS
- **CPU:** 2 cores
- **RAM:** 2GB
- **Disk:** 50GB on the `vm-storage` pool (Samsung 860 EVO)
- **Network:** VirtIO, bridged via vmbr0

## Setup Notes

- Installed via Proxmox's "Download from URL" feature, pulling the ISO directly onto Ironhide without needing physical media
- Configured with Q35 machine type and OVMF (UEFI) firmware
- QEMU Guest Agent installed and enabled, allowing Proxmox to see the VM's IP address, perform clean shutdowns, and take consistent snapshots
- OpenSSH server installed during setup for remote terminal access

## Access

Initially only reachable via the Proxmox console or local network SSH. Installing Tailscale directly inside the VM gave it its own independent network identity, making it directly SSH-accessible from anywhere without needing to go through Proxmox at all.

## Current Use

Currently a clean, general-purpose server with no services running yet — the next phase involves deploying actual workloads (web server, Git hosting, or containerized apps) as a hands-on extension of coursework.
