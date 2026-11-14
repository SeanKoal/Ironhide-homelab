# Wazuh SIEM

Wazuh collects security logs from every machine in the lab into one place, flags suspicious activity, and maps it to MITRE ATT&CK techniques.

## Why

Before Wazuh, every machine kept its own logs, and nobody was reading them. If an attacker got into a machine, they could also wipe its logs. Wazuh copies events off each machine as they happen, so there's a central record that survives even if a machine is compromised.

## Setup

| Setting | Value |
|---|---|
| VM | 103 `wazuh`, Ubuntu Server 24.04 |
| Address | `192.168.2.80` |
| Version | Wazuh 4.14 |
| Install type | All in one (server, indexer, and dashboard on the same VM) |
| Dashboard | `https://192.168.2.80` |

Installed with Wazuh's official all in one script:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The admin password the installer generates is stored in Bitwarden.

The VM starts automatically with Ironhide (boot order 4, after the other VMs) and is included in the nightly backup job.

## Log Sources

| Source | Method | What it sends |
|---|---|---|
| Ubuntu Server (100) | Wazuh agent | Auth logs, system logs, file integrity, rootcheck |
| Kali (101) | Wazuh agent | Same as above |
| Ironhide (Proxmox host) | Wazuh agent | Same as above |
| TrueNAS (102) | Syslog over UDP 514 | TrueNAS system logs |

**Why syslog for TrueNAS:** TrueNAS is an appliance, and installing extra software on it isn't supported. It can forward its logs over syslog, set under System, then Advanced, then Syslog in the TrueNAS UI.

The Wazuh manager accepts syslog only from TrueNAS. This block was added to `/var/ossec/etc/ossec.conf`:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>192.168.2.136</allowed-ips>
</remote>
```

Then restarted the manager:

```bash
sudo systemctl restart wazuh-manager
```

## Testing It: Simulated SSH Brute Force

To confirm detection actually works, Kali attacked Ubuntu Server with repeated SSH logins as a user that doesn't exist:

```bash
ssh fakeuser@192.168.2.35
```

**What happened on the target:** fail2ban banned Kali's IP after the failed attempts, and further connections hung.

**What Wazuh recorded** (Threat Hunting, Events, filtered to the Ubuntu Server agent):

| Rule | Level | Description |
|---|---|---|
| 5710 | 5 | sshd: Attempt to login using a non-existent user |
| 5503 | 5 | PAM: User login failed |
| 2502 | 10 | User missed the password more than one time |

Each failed login on its own is a low level event, since random failed logins are normal internet noise. Rule 2502 is the important one. Wazuh noticed the failures repeating from the same source and escalated to level 10. That correlation is what separates a SIEM from just reading logs. The source IP in the alerts was `192.168.2.244` (Kali), and it maps to MITRE ATT&CK **T1110, Brute Force**.

**Result:** Two layers worked together. fail2ban blocked the attack on the server, and Wazuh kept a central record and raised the severity.

## Other Alerts Seen After Install

- **First time user executed sudo (5403):** caused by installing the agent. Harmless here, but exactly what should be flagged if it wasn't expected.
- **PAM login session opened/closed:** normal SSH logins.
- **Rootcheck (510) on Ubuntu Server:** Wazuh's rootkit scan flagging things on the Docker host. Containers do things that look suspicious to a rootkit scanner, so these are likely false positives. Needs tuning.

## Notes

- The agent on Ubuntu Server peaked around 645 MB of RAM on a 2 GB VM that also runs the website and Docker. Worth watching.

## Still To Do

- Tune out the Docker rootcheck false positives
- Rotate the dashboard admin password with the Wazuh password tool
- Reserve `.80` for the Wazuh VM on the router
