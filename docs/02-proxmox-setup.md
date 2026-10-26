Proxmox Setup
Installation

Proxmox VE was installed fresh onto the WD Blue SN570 NVMe drive, serving as the dedicated boot/OS drive separate from VM storage.

Initial Configuration
Fixed Proxmox's default enterprise repository, switching to the no-subscription community repo to allow updates without a paid license
Confirmed local network access via the Proxmox web UI
BIOS Optimization
XMP enabled — RAM was running at the default 2133MHz out of the box; enabling A-XMP Profile 1 in BIOS brought it up to its rated 3200MHz
SVM Mode (AMD-V) enabled — required for hardware virtualization, allowing Proxmox to properly run VMs
BIOS update pending. Currently on E7B86AMS.H60 (April 2020), target is E7B86AMS.1J9 via M-Flash. A BIOS update resets settings to default, so XMP and SVM Mode must be re-enabled right after flashing.
Display Output

The Ryzen 5 3600 has no integrated graphics, and the MSI B450 Gaming Plus will not complete POST without a GPU installed. A dedicated EVGA GT 710 stays in Ironhide permanently so the host always boots on its own and has a local console available for BIOS access and troubleshooting. It needs no external power and draws very little.

Storage Configuration
Samsung 860 EVO 1TB configured as an LVM-Thin pool (vm-storage), used as the primary storage for all VM disks
Seagate IronWolf 2TB intentionally left unformatted at the Proxmox level — passed through directly to the TrueNAS VM later so TrueNAS has full raw control of the disk, rather than inheriting a filesystem Proxmox had already touched
Host Networking
Setting	Value
Host IP	192.168.2.10 on bridge vmbr0, reserved by MAC on the ASUS router
Gateway	192.168.2.1 (ASUS RT-AC68U)
/etc/hosts	192.168.2.10 ironhide.local ironhide
DNS	192.168.2.1 primary, 1.1.1.1 backup, set directly in /etc/resolv.conf
Timezone	America/New_York

Tailscale's DNS management is turned off on the host (tailscale set --accept-dns=false). Left on, it rewrites /etc/resolv.conf to point at its own resolver, which broke all internet name lookups on this host. The host only needs normal DNS.

Package Repositories

Proxmox VE 9 runs on Debian trixie, so every repo must match that release.

Proxmox: /etc/apt/sources.list.d/proxmox.sources, trixie pve-no-subscription
Debian: trixie, trixie-updates, trixie-security
Tailscale: official Tailscale repo
Enterprise and Ceph: present but commented out, since there is no paid subscription
VM Auto-Start and Boot Order

All VMs start automatically when Ironhide boots, in a set order so dependencies come up first:

Order	VMID	VM	Why
1	102	TrueNAS	Storage comes up first
2	100	Ubuntu Server	Hosts the public website
3	101	Kali	Least critical

Set with:

bash
qm set 102 --onboot 1 --startup order=1
qm set 100 --onboot 1 --startup order=2
qm set 101 --onboot 1 --startup order=3
Updating the Host
bash
apt update
apt full-upgrade
reboot

Proxmox uses full-upgrade rather than plain upgrade, since its updates sometimes need to add or replace packages. A reboot is needed whenever a new kernel is installed. With auto-start configured, all VMs come back on their own.

Current as of October 2026: Proxmox VE 9.2.21, kernel 7.0.14, Tailscale 1.104.2.

Remote Access
Tailscale installed directly on Ironhide, giving it a persistent private mesh network identity reachable securely from anywhere with an internet connection
This became the foundation for all remote homelab work — every VM built afterward was also given its own independent Tailscale identity, allowing direct access without needing to route through Proxmox
For host maintenance that restarts Tailscale itself, connect over the local address (ssh root@192.168.2.10) instead, so the session doesn't drop mid-update
Notes
The Proxmox subscription nag popup was investigated but determined to be a cosmetic-only issue with no functional impact; not worth pursuing further given the JavaScript patch didn't apply cleanly on this Proxmox version (9.x)
