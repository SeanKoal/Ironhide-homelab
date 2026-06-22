# Troubleshooting Log

Real problems encountered while building this homelab, documented as they were solved. Less polished than the other docs on purpose — this is meant to capture actual debugging process.

---

## Kali VNC: Blank Grey Screen

**Symptom:** Connecting via VNC showed only a grey screen with no desktop rendering.

**Investigation:**
- Checked the VNC server log, found Xfce's window manager (`xfwm4`) failing with errors like "Cannot find visual format on screen 0" and a long list of missing X extensions (XRender, XRandr, XComposite, etc.)
- Root cause: the default `tightvncserver` package is an unmaintained implementation from 2009, incapable of providing a modern enough virtual display for current Xfce to render properly

**Fix:** Replaced `tightvncserver` with `tigervnc-standalone-server`, an actively maintained alternative.

---

## Kali VNC: Connection Refused

**Symptom:** Even after switching to TigerVNC, connections were refused entirely.

**Investigation:**
- Ran `netstat -tulnp | grep 5901` and found VNC was listening only on localhost only, invisible to any external connection including Tailscale

**Fix:** Restarted VNC explicitly with `-localhost no` to bind it to all network interfaces.

---

## Kali VNC: Systemd Service Wouldn't Start

**Symptom:** A custom systemd service for auto-starting VNC on boot repeatedly failed with confusing errors ("X11 server already running," "bad unit file setting").

**Investigation:**
- Several layered issues stacked on top of each other: leftover lock files from previous manual VNC sessions, incorrect `Type=forking` vs the server's actual behavior, and finally a close read of the unit file revealing lowercase section headers (`[service]` instead of `[Service]`) and a typo'd binary path (`/usr/vncserver` instead of `/usr/bin/vncserver`)

**Fix:** Rewrote the systemd unit file carefully with correct capitalization and paths, cleared all leftover lock files, and used `Type=simple` with the `-fg` flag to match how TigerVNC actually runs.

---

## TrueNAS: Locked Out After Install

**Symptom:** Neither the "admin" account created during install nor the root account could log into the TrueNAS web UI — both rejected as incorrect credentials.

**Investigation:**
- Queried TrueNAS's user database directly via its internal CLI tool (`midclt call user.query`)
- Discovered no admin account had actually been created despite selecting that option during install
- Found root's account showed `"password_disabled": true` — authentication was explicitly disabled at the TrueNAS application layer, separate from any OS-level password lock

**Fix:** Used `midclt call user.update` to directly set a new root password and flip `password_disabled` to false through TrueNAS's own management API, bypassing the broken web UI path entirely.

---

## TrueNAS: Pool Creation Failed on Duplicate Serial

**Symptom:** Creating a storage pool failed with "Disks have duplicate serial numbers: None (sda, sdb)."

**Investigation:**
- Both the OS disk and the passed-through IronWolf were reporting blank ("None") serial numbers in this virtualized context, causing TrueNAS to treat them as indistinguishable

**Fix:** Re-attached the IronWolf at the Proxmox level with an explicitly assigned serial number, then rebooted the VM so TrueNAS could see the corrected disk metadata.

---

---

## ASUS RT-AC68U: Factory Reset Wouldn't Take

**Symptom:** After a standard factory reset (holding the reset button while powered on), the router's WiFi never came back, the power LED kept blinking indefinitely, and previously-configured custom SSIDs ("The Matrix," "Johnny Mnemonic" — leftover from the router's prior owner) briefly disappeared and reappeared inconsistently across multiple reset attempts.

**Investigation:**
- A standard reset attempt left the router in an unstable state — WAN light active, but no WiFi broadcasting and wired LAN ports flapping (connecting briefly, then dropping)
- A full power cycle (unplug, wait 30 seconds, replug) was required in addition to the reset itself before WiFi finally returned, broadcasting under the correct factory defaults ("ASUS" / "ASUS_5G") found printed on the physical product label

**Fix:** Combined a standard factory reset with a full power cycle, then allowed several minutes of uninterrupted boot time before evaluating the router's state. Confirmed success only once the default factory SSIDs appeared, rather than assuming a reset had succeeded based on LED behavior alone.

---

## ASUS RT-AC68U: Internet Status Showed "IP Conflict Detected"

**Symptom:** After successfully resetting and reconfiguring the router with new SSIDs and a WAN connection physically wired to a secondary modem/router, the dashboard reported "Internet status: IP conflict detected," despite a confirmed working physical connection on both ends.

**Investigation:**
- Verified the physical link was good by testing the cable and both ports independently (swapping cables, testing ports directly with a laptop)
- Determined the issue was logical, not physical: the ASUS's default LAN IP (192.168.1.1) was identical to the upstream router's LAN IP, causing an address collision between the two devices

**Fix:** Changed the ASUS's LAN IP address to a different subnet, resolving the conflict immediately and bringing the internet connection online.

---

## Lessons Learned

- When troubleshooting secondhand networking hardware, don't trust LED status alone — a device can appear "on" and partially functional while still being in a broken or incomplete internal state
- Two routers on the same local network will conflict if they share default IP ranges; this is a common and easily overlooked issue when introducing a second router behind an existing one
- When a service fails mysteriously, check the actual logs before guessing — `journalctl`, application-specific log files, and direct API queries (like TrueNAS's `midclt`) consistently revealed the real cause faster than trial and error
- Virtualized environments can introduce quirks (blank disk serials, localhost-only binding) that wouldn't occur on bare metal — always verify assumptions about networking and hardware identification in a VM context
- Outdated packages (like the old TightVNC implementation) can cause problems that look like configuration errors but are actually compatibility issues — worth checking if a more modern alternative exists
