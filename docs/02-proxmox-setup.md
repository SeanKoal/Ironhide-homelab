# Proxmox Setup

## Installation

Proxmox VE was installed fresh onto the WD Blue SN570 NVMe drive, serving as the dedicated boot/OS drive separate from VM storage.

## Initial Configuration

- Fixed Proxmox's default enterprise repository, switching to the no-subscription community repo to allow updates without a paid license
- Confirmed local network access via the Proxmox web UI

## BIOS Optimization

- **XMP enabled** — RAM was running at the default 2133MHz out of the box; enabling A-XMP Profile 1 in BIOS brought it up to its rated 3200MHz
- **SVM Mode (AMD-V) enabled** — required for hardware virtualization, allowing Proxmox to properly run VMs

## Storage Configuration

- **Samsung 860 EVO 1TB** configured as an LVM-Thin pool (`vm-storage`), used as the primary storage for all VM disks
- **Seagate IronWolf 2TB** intentionally left unformatted at the Proxmox level — passed through directly to the TrueNAS VM later so TrueNAS has full raw control of the disk, rather than inheriting a filesystem Proxmox had already touched

## Remote Access

- Tailscale installed directly on Ironhide, giving it a persistent private mesh network identity reachable securely from anywhere with an internet connection
- This became the foundation for all remote homelab work — every VM built afterward was also given its own independent Tailscale identity, allowing direct access without needing to route through Proxmox

## Notes

- The Proxmox subscription nag popup was investigated but determined to be a cosmetic-only issue with no functional impact; not worth pursuing further given the JavaScript patch didn't apply cleanly on this Proxmox version (9.2.2)
