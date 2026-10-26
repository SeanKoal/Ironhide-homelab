Troubleshooting Log

Real problems encountered while building this homelab, documented as they were solved. Less polished than the other docs on purpose — this is meant to capture actual debugging process.

Kali VNC: Blank Grey Screen

Symptom: Connecting via VNC showed only a grey screen with no desktop rendering.

Investigation:

Checked the VNC server log, found Xfce's window manager (xfwm4) failing with errors like "Cannot find visual format on screen 0" and a long list of missing X extensions (XRender, XRandr, XComposite, etc.)
Root cause: the default tightvncserver package is an unmaintained implementation from 2009, incapable of providing a modern enough virtual display for current Xfce to render properly

Fix: Replaced tightvncserver with tigervnc-standalone-server, an actively maintained alternative.

Kali VNC: Connection Refused

Symptom: Even after switching to TigerVNC, connections were refused entirely.

Investigation:

Ran netstat -tulnp | grep 5901 and found VNC was listening only on localhost only, invisible to any external connection including Tailscale

Fix: Restarted VNC explicitly with -localhost no to bind it to all network interfaces.

Kali VNC: Systemd Service Wouldn't Start

Symptom: A custom systemd service for auto-starting VNC on boot repeatedly failed with confusing errors ("X11 server already running," "bad unit file setting").

Investigation:

Several layered issues stacked on top of each other: leftover lock files from previous manual VNC sessions, incorrect Type=forking vs the server's actual behavior, and finally a close read of the unit file revealing lowercase section headers ([service] instead of [Service]) and a typo'd binary path (/usr/vncserver instead of /usr/bin/vncserver)

Fix: Rewrote the systemd unit file carefully with correct capitalization and paths, cleared all leftover lock files, and used Type=simple with the -fg flag to match how TigerVNC actually runs.

TrueNAS: Locked Out After Install

Symptom: Neither the "admin" account created during install nor the root account could log into the TrueNAS web UI — both rejected as incorrect credentials.

Investigation:

Queried TrueNAS's user database directly via its internal CLI tool (midclt call user.query)
Discovered no admin account had actually been created despite selecting that option during install
Found root's account showed "password_disabled": true — authentication was explicitly disabled at the TrueNAS application layer, separate from any OS-level password lock

Fix: Used midclt call user.update to directly set a new root password and flip password_disabled to false through TrueNAS's own management API, bypassing the broken web UI path entirely.

TrueNAS: Pool Creation Failed on Duplicate Serial

Symptom: Creating a storage pool failed with "Disks have duplicate serial numbers: None (sda, sdb)."

Investigation:

Both the OS disk and the passed-through IronWolf were reporting blank ("None") serial numbers in this virtualized context, causing TrueNAS to treat them as indistinguishable

Fix: Re-attached the IronWolf at the Proxmox level with an explicitly assigned serial number, then rebooted the VM so TrueNAS could see the corrected disk metadata.

ASUS RT-AC68U: Factory Reset Wouldn't Take

Symptom: After a standard factory reset (holding the reset button while powered on), the router's WiFi never came back, the power LED kept blinking indefinitely, and previously-configured custom SSIDs ("The Matrix," "Johnny Mnemonic" — leftover from the router's prior owner) briefly disappeared and reappeared inconsistently across multiple reset attempts.

Investigation:

A standard reset attempt left the router in an unstable state — WAN light active, but no WiFi broadcasting and wired LAN ports flapping (connecting briefly, then dropping)
A full power cycle (unplug, wait 30 seconds, replug) was required in addition to the reset itself before WiFi finally returned, broadcasting under the correct factory defaults ("ASUS" / "ASUS_5G") found printed on the physical product label

Fix: Combined a standard factory reset with a full power cycle, then allowed several minutes of uninterrupted boot time before evaluating the router's state. Confirmed success only once the default factory SSIDs appeared, rather than assuming a reset had succeeded based on LED behavior alone.

ASUS RT-AC68U: Internet Status Showed "IP Conflict Detected"

Symptom: After successfully resetting and reconfiguring the router with new SSIDs and a WAN connection physically wired to a secondary modem/router, the dashboard reported "Internet status: IP conflict detected," despite a confirmed working physical connection on both ends.

Investigation:

Verified the physical link was good by testing the cable and both ports independently (swapping cables, testing ports directly with a laptop)
Determined the issue was logical, not physical: the ASUS's default LAN IP (192.168.1.1) was identical to the upstream router's LAN IP, causing an address collision between the two devices

Fix: Changed the ASUS's LAN IP address to a different subnet, resolving the conflict immediately and bringing the internet connection online.

Network Migration: Ironhide Lost Connectivity After Moving to the ASUS Network

Symptom: After physically moving Ironhide's ethernet cable from the original network to the newly configured ASUS standalone network, Ironhide became unreachable both locally and via Tailscale, despite a healthy physical link light on the new connection.

Investigation:

Confirmed the physical link was active and the correct new IP was assigned to the physical interface (nic0) after manually forcing a DHCP renewal
Despite the correct IP on nic0, the host still could not reach its own gateway or the internet
Checked ip route show and found a stale default route still pointing to the old gateway — manually deleting it and forcing a fresh DHCP renewal corrected the route, but connectivity still failed
The actual root cause was found in /etc/network/interfaces: Proxmox's network bridge (vmbr0) — which the host and all VMs actually communicate through — was hardcoded with a static IP and gateway from the old network, completely independent of whatever the physical interface (nic0) was doing

Fix: Edited /etc/network/interfaces directly, updating the vmbr0 bridge's static address and gateway lines to match the new network, then restarted networking. This was the actual fix — nic0's configuration alone was never going to resolve it, since the bridge is what the host and VMs genuinely depend on.

Network Migration: VMs Stayed Offline on Tailscale After the Host Recovered

Symptom: After fixing Ironhide's bridge configuration and restoring host-level connectivity, all three VMs (Ubuntu Server, Kali, TrueNAS) remained offline on Tailscale, each still showing their old, stale IP addresses internally.

Investigation:

Checked each VM's network configuration via qm config <vmid> on the Proxmox host and found every VM had firewall=1 set on its network device — Proxmox's per-VM firewall flag, separate from the host-level firewall (which had already been confirmed disabled)
This per-VM firewall flag was blocking DHCP traffic for each VM individually, preventing them from acquiring a new lease on the new network even after the host itself was healthy

Fix:

Disabled the per-VM firewall flag on all three VMs using qm set <vmid> --net0 virtio=<mac>,bridge=vmbr0,firewall=0
Even after that fix, each VM was still holding onto its old, stale IP lease and required a manual forced renewal — ip addr flush dev <interface> followed by dhcpcd <interface> (or dhclient, depending on the distro) — rather than recovering automatically
One VM (Kali) ended up holding both the old and new IP simultaneously after renewal; the stale address had to be manually removed with ip addr del <old-ip> dev <interface> before things resolved cleanly
Ironhide: Host Silently Unable to Update for Months

Symptom: While running a security patching pass in October 2026, apt update on the Proxmox host failed on every single repository with "Temporary failure resolving." The host had 187 packages waiting, and it had likely been unable to pull updates since the move to the ASUS network.

Investigation:

ping 1.1.1.1 worked and ip route showed the correct gateway (192.168.2.1), so the host had internet access. The problem was name resolution only.
/etc/resolv.conf was generated by Tailscale and pointed at its MagicDNS resolver (100.100.100.100). MagicDNS forwards normal lookups to whatever DNS the system used before Tailscale took over, which was left over from the old network. Every lookup hit a dead end.
Running tailscale set --accept-dns=false handed the file back, but it came back empty. There was no valid DNS configuration underneath at all.

Fix: Disabled Tailscale DNS management on the host, then wrote /etc/resolv.conf by hand with the ASUS router (192.168.2.1) as primary and Cloudflare (1.1.1.1) as backup. Confirmed with ping deb.debian.org and verified the file survived a full reboot. A Proxmox host only needs normal internet DNS, not tailnet name lookups.

Proxmox: Package Repository Pointed at the Wrong Debian Release

Symptom: Once DNS was working, apt update showed the Debian repos on trixie but the Proxmox repo on bookworm.

Investigation:

pveversion confirmed the host runs Proxmox VE 9, which is built on Debian trixie
/etc/apt/sources.list still contained a Proxmox 8 pve-no-subscription line for bookworm. Mixing release versions is a common way to break an upgrade.
The enterprise and Ceph repos were correctly commented out, since there is no paid subscription

Fix: Backed up sources.list, removed the bookworm line with sed, and created a new /etc/apt/sources.list.d/proxmox.sources file in the newer deb822 format pointing at the trixie pve-no-subscription repo. apt update then ran clean, and apt full-upgrade applied all 187 updates, including a new kernel, Proxmox VE 9.2.21, and Tailscale 1.104.2.

Ironhide: Stale Leftovers from the Network Migration

Symptom: While checking service health, the pve-cluster log showed it resolving the node name ironhide to 192.168.1.208, an address from the original network. The host's real address is 192.168.2.10.

Investigation:

/etc/hosts still held the original install's address. Proxmox's cluster services use this entry to identify the node, and a stale value is a known cause of hung management services.
The same pass also turned up the system timezone set to America/Adak (Aleutian Islands), which made every log timestamp misleading.

Fix: Backed up and corrected /etc/hosts to 192.168.2.10, and set the timezone to America/New_York with timedatectl. Both confirmed after reboot.

Ubuntu Server: cloudflared Stuck on an Old Version

Symptom: A Cloudflare security notice asked all Tunnel users to upgrade cloudflared to 2026.10.2 or later after eight eMudhra intermediate certificates were found to have trivially recoverable private keys. The server was running 2026.6.1.

Investigation:

Did not act on links in the email. Verified the issue independently through published security reporting first.
apt policy cloudflared showed the only source was /var/lib/dpkg/status, meaning it was originally installed from a downloaded .deb. Apt had never been able to update it.

Fix: Added Cloudflare's official signed apt repository, then upgraded through apt to 2026.10.4 alongside 60 other pending packages. Confirmed the tunnel reconnected (systemctl status cloudflared) and the site stayed live. Future versions now arrive through normal updates.

Lessons Learned
A host can have a perfectly correct IP on its physical network interface while still being completely unreachable — Proxmox's virtual bridge (vmbr0) is what actually carries traffic for the host and all VMs, and it can hold its own independent, stale configuration separate from the physical interface
Proxmox firewalls exist at two separate levels — host-wide and per-VM — and disabling one does not affect the other; both need to be checked independently when diagnosing connectivity issues
After any major network topology change (new subnet, new gateway), don't assume devices will automatically renew their DHCP leases — Linux network managers often hold onto stale configuration until explicitly forced to flush and request a new lease
A network migration leaves old settings scattered in places that don't break right away. The bridge config failed loudly, but /etc/hosts and DNS kept quietly pointing at the old network for months. After any move, audit every place an address can live.
Tailscale's MagicDNS takes over /etc/resolv.conf. On servers that don't need tailnet name lookups, turn it off with --accept-dns=false and set DNS directly.
When troubleshooting secondhand networking hardware, don't trust LED status alone — a device can appear "on" and partially functional while still being in a broken or incomplete internal state
Two routers on the same local network will conflict if they share default IP ranges; this is a common and easily overlooked issue when introducing a second router behind an existing one
When a service fails mysteriously, check the actual logs before guessing — journalctl, application-specific log files, and direct API queries (like TrueNAS's midclt) consistently revealed the real cause faster than trial and error
Virtualized environments can introduce quirks (blank disk serials, localhost-only binding) that wouldn't occur on bare metal — always verify assumptions about networking and hardware identification in a VM context
Outdated packages (like the old TightVNC implementation) can cause problems that look like configuration errors but are actually compatibility issues — worth checking if a more modern alternative exists
Software installed from a downloaded package never updates itself. Install from the vendor's official repository so security fixes arrive through normal updates.
Treat security emails as a lead, not an instruction. Verify the issue independently, then patch through trusted channels.
