# Networking

## Overview

The homelab is built around Tailscale, a mesh VPN that gives every device — the server, each VM, laptops, and even a phone — its own private, encrypted, directly reachable identity, regardless of physical location.

## Why Tailscale

Traditional remote access setups require port forwarding and exposing services to the public internet, which carries real security risk. Tailscale instead builds a private encrypted mesh network between only the devices you authorize, with no inbound ports ever exposed publicly.

## Devices on the Network

- **Ironhide** — the Proxmox host itself
- **Ubuntu Server VM** — independent Tailscale identity, SSH accessible directly
- **Kali Linux VM** — independent Tailscale identity, SSH + VNC desktop access
- **TrueNAS VM** — independent Tailscale identity, installed via TrueNAS's containerized Apps system (since TrueNAS SCALE runs an immutable base OS that blocks direct package installation)
- **Soundwave** — primary remote access laptop
- **Personal phone** — for on-the-go monitoring and access

## Key Lesson: VMs Don't Need to Route Through the Host

Initially, VMs were only reachable through the Proxmox console once Tailscale was set up on Ironhide itself. Installing Tailscale **inside each VM individually** gave them independent network identities — meaning Ubuntu Server, Kali, and TrueNAS are each directly reachable via SSH, VNC, or web UI from anywhere, with no dependency on Proxmox being open at all.

## Local Network

Ironhide also remains reachable on the local home network for situations where Tailscale isn't available or for local-only access. Local network access is intentionally kept separate from broader home network exposure — for example, TrueNAS is currently accessed exclusively over Tailscale rather than configuring local network shares, since other devices/users share the home network and shouldn't have access to homelab storage.

## Future Plans

A secondhand ASUS RT-AC68U router has been acquired and is planned for future use, likely as a dedicated access point or as part of a more advanced network segmentation setup (potentially alongside VLANs once a managed switch is added to the homelab).
