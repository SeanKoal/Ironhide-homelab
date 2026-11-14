# Backups and Recovery

Every VM that matters gets backed up nightly from Proxmox to TrueNAS, and the restore process has been tested with a real disaster drill.

## How It Works

Proxmox's built-in backup tool (`vzdump`) writes VM backups to a share on TrueNAS over NFS. The backups land on the IronWolf drive, which is a separate physical disk from the SSD the VMs run on. If the VM SSD dies, the backups survive.

| Piece | Value |
|---|---|
| TrueNAS dataset | `/mnt/ironwolf-pool/proxmox-backups` |
| Share type | NFS, restricted to Ironhide only (`192.168.2.10`) |
| Proxmox storage name | `truenas-backups` (content type: Backup only) |

## The Backup Job

Set under Datacenter, then Backup, in the Proxmox web UI.

| Setting | Value | Why |
|---|---|---|
| VMs | 100 (Ubuntu Server), 101 (Kali), 103 (Wazuh) | |
| Schedule | Daily at 03:00 | Nothing is in use overnight |
| Mode | Snapshot | VMs stay running during the backup |
| Compression | zstd | Fast with a good compression ratio |
| Retention | 7 daily, 4 weekly | About a month of history without filling the drive |

Equivalent command:

```bash
vzdump 100 101 103 --compress zstd --mode snapshot --storage truenas-backups --prune-backups 'keep-daily=7,keep-weekly=4'
```

The QEMU guest agent runs inside the VMs, so Proxmox can briefly freeze the filesystem during a snapshot. This gives a clean, consistent backup instead of one taken mid write.

**TrueNAS (102) is not in the job on purpose.** It stores the backups, and its data drive is passed through raw, so backing it up to itself would be pointless. Its config can be exported from the TrueNAS UI instead.

## Checking Backups

```bash
pvesm list truenas-backups
```

Each VM shows one `vzdump-qemu-<vmid>-<date>.vma.zst` file per backup.

## Taking a Backup Before a Risky Change

Before any planned change that could break a VM, take a fresh backup first:

```bash
vzdump 101 --storage truenas-backups --mode snapshot --compress zstd
```

## Restoring a VM

1. Stop the VM
2. Open the VM's **Backup** tab and switch the **Storage** dropdown to `truenas-backups` (it defaults to a different storage, so the backups look missing at first)
3. Select the backup and click **Restore**
4. Confirm the overwrite warning, tick **Start after restore**
5. Wait for **TASK OK** in the task log

## Disaster Drill (November 2026)

A backup that has never been restored is only a guess. This drill proved the restore process works end to end.

**Setup:**
- Took a fresh backup of Kali (VM 101) at 4:00 PM so the Wazuh agent installed earlier that day would be included
- Created a marker file (`~/after-backup.txt`) on Kali after the backup. If the restore truly rolled back, this file would disappear.

**Break:** Deleted everything in `/boot` with `sudo rm -rf /boot/*` and rebooted. With no kernel or bootloader files, Kali could no longer boot.

**Recover:** Force stopped VM 101 and restored the 4:00 PM backup from `truenas-backups`.

**Verify:**
- Kali booted normally and `/boot` was back
- `after-backup.txt` was gone, proving the VM really rolled back to the backup
- Kali's Wazuh agent reconnected and showed as active

**Results:**

| Metric | Result | Meaning |
|---|---|---|
| RPO (recovery point objective) | About 5 minutes | How much work would be lost: the age of the newest backup when the failure hit |
| RTO (recovery time objective) | About 4 minutes | How long it took from failure to a working system |

With only the nightly job, RPO in a real failure could be up to 24 hours. That's acceptable for a homelab. Anything more important gets a manual backup before risky work.

## Still To Do

- Copy backups off Ironhide entirely (to Magnus' 1TB HDD) so a total hardware loss doesn't take the backups with it
- Add a second IronWolf as a ZFS mirror so a single drive failure doesn't lose the backup pool
