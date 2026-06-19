# TrueNAS VM

## Purpose

A dedicated Network Attached Storage system, managing the Seagate IronWolf 2TB drive. Intended as centralized, private storage accessible only by me, with future plans to add network shares and self-hosted apps.

## Specs

- **OS:** TrueNAS SCALE 25.10.4
- **CPU:** 2 cores
- **RAM:** 8GB
- **OS Disk:** 32GB on the `vm-storage` pool
- **Storage Disk:** Seagate IronWolf 2TB, passed through directly as a raw physical disk (not virtualized), giving TrueNAS full native control

## Setup Notes

### Disk Passthrough

Rather than creating a virtual disk for storage, the IronWolf was passed through directly using its unique disk ID via the Proxmox shell, so TrueNAS manages the physical drive natively rather than inheriting a Proxmox-managed filesystem layer.

### Authentication Issue

After installation, neither the configured "admin" account nor the root account could log into the web UI. Investigation revealed:
- The installer's "admin user" option had not actually created a separate admin account
- The root account existed but had password authentication explicitly disabled at the TrueNAS application layer (`password_disabled: true`), separate from any OS-level lock

Resolved by using TrueNAS's internal management API directly (`midclt`) to set a root password and re-enable password authentication.

### Disk Serial Collision

Pool creation initially failed with a "duplicate serial number" error, caused by both the OS disk and the passed-through IronWolf reporting blank serial numbers in this virtualized context. Fixed by explicitly assigning a serial number to the passthrough disk at the Proxmox level.

## Storage Pool

- **Pool name:** `ironwolf-pool`
- **Layout:** Single-disk stripe (no redundancy — by necessity, since only one drive is currently available)
- **Usable capacity:** ~1.82 TiB

## Access

- Web UI and Tailscale installed via TrueNAS SCALE's containerized **Apps** system, since the underlying OS is intentionally immutable/read-only and blocks direct package installation
- Currently accessed exclusively via Tailscale; no local network shares configured yet, keeping storage private to a single authorized user

## Current Use

Storage pool is live and ready. Next steps include setting up authenticated network shares (SMB) for actual file access from Magnus and Soundwave, and exploring TrueNAS's app ecosystem.
