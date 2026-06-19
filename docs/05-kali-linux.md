# Kali Linux VM

## Purpose

A dedicated security and penetration testing environment, pre-loaded with industry-standard tools. Used for hands-on practice directly tied to CompTIA Security+ studies and upcoming security coursework.

## Specs

- **OS:** Kali Linux 2026.1, GUI installer (Xfce desktop environment)
- **CPU:** 2 cores
- **RAM:** 4GB (bumped up from the Ubuntu Server baseline to comfortably run a full desktop environment)
- **Disk:** 50GB on the `vm-storage` pool
- **Network:** VirtIO, bridged via vmbr0

## Setup Notes

- Installed with SeaBIOS instead of OVMF/UEFI — the UEFI boot selection screen had an unresponsive Enter key issue specific to this VM, resolved by switching firmware types
- QEMU Guest Agent installed for clean shutdowns and accurate resource reporting
- SSH server enabled manually post-install, since Kali (unlike Ubuntu Server) doesn't enable it by default

## Remote Desktop Access (VNC)

Getting full GUI access working remotely required solving several layered issues:

1. The default `tightvncserver` package (a very outdated 2009-era implementation) couldn't properly render the modern Xfce desktop, resulting in a blank grey screen
2. Switched to **TigerVNC**, a modern, actively maintained VNC server
3. The `xstartup` script needed adjusting to properly hand off the display to Xfce
4. VNC was initially bound only to `localhost`, silently
