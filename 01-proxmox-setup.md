# 1. Installing Proxmox VE

The first step was turning the OptiPlex into a hypervisor that could run several VMs and be managed from a web browser. I chose Proxmox VE because it's free, runs directly on the hardware, has a full web interface, and is widely used in homelabs and small businesses.

**End result:** Proxmox VE 9.2.2 running at `https://SERVER_IP:8006`, fully updated.

## BIOS Settings

Before installing, I changed these settings in the BIOS:

| Setting | Value | Reason |
|---|---|---|
| Virtualization (VT-x) | Enabled | Required to run VMs |
| VT for Direct I/O (VT-d) | Enabled | Allows hardware passthrough later |
| AC Recovery | Power On | The server turns back on after a power outage |
| Wake on LAN | Enabled | Lets me power it on over the network |
| Secure Boot | Disabled | Avoids boot problems with the installer |

I connected the server with an Ethernet cable rather than Wi-Fi, since Proxmox needs a wired connection to bridge VMs onto the network properly.

## Installation

1. Downloaded the Proxmox VE installer ISO and flashed it to a USB drive.
2. Booted from the USB using the F12 boot menu.
3. Installed with these choices:
   - **Disk:** internal SSD
   - **Filesystem:** ext4 (ZFS uses a lot of RAM and doesn't add much with a single drive)
   - **Hostname:** `pve.home.lan`
   - **IP address:** static

## Problem: Web Interface Timed Out

After installing, I tried to open the web interface from my PC and the browser timed out.

To figure out why, I ran `ipconfig` on my PC:

```
IPv4 Address . . . : PC_IP
Default Gateway  . : GATEWAY_IP
```

My home network was on a different subnet than the static IP I had given Proxmox during the install (`192.168.1.50`). Because they were on different networks, my PC had no way to reach the server.

**Fix:** I edited the network configuration on the server to use an address on the correct subnet.

In `/etc/network/interfaces`:
```
iface vmbr0 inet static
        address SERVER_IP/24
        gateway GATEWAY_IP
```

I also updated `/etc/hosts` and `/etc/resolv.conf` to match, then restarted networking and confirmed I could reach the router:

```bash
systemctl restart networking
ping -c 3 GATEWAY_IP
```

After that, the web interface loaded. I also set a DHCP reservation in my router so no other device can be given the server's address.

**What I learned:** a static IP has to be inside the same subnet as the network the machine is plugged into. Checking the gateway first would have avoided the problem.

![Proxmox dashboard](images/01-proxmox-dashboard.png)

## Problem: Updates Failing

The "Update package database" task failed with an error. The repository page showed that the enterprise repository was enabled without a subscription.

**Cause:** By default, Proxmox points to its paid enterprise update servers, which reject systems without a subscription.

**Fix:** Under Updates > Repositories, I:
1. Disabled the `pve-enterprise` and `ceph` enterprise repositories
2. Added the free `No-Subscription` repository
3. Refreshed and installed all available updates

![Repository configuration](images/01-proxmox-repositories.png)

## Result

- Proxmox VE 9.2.2 installed and fully updated
- Static IP on the correct subnet, reserved in the router
- Server powers back on automatically after an outage
