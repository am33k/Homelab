# 2. Secure Remote Access with Tailscale

I wanted to reach the server and all my VMs from school or anywhere else, with full desktop access and not just a terminal. I also didn't want to expose anything to the internet.

**End result:** Every machine in the lab is reachable over an encrypted VPN, with no ports opened on my router.

## Why Not Port Forwarding

The simple option would be to forward ports like 3389 (RDP) or 8006 (Proxmox) on my router. That would put those services directly on the internet, where they would constantly be scanned and targeted by automated attacks.

Instead, I used **Tailscale**, a VPN built on WireGuard. Only devices signed in to my account can connect, all traffic is encrypted, and it only makes outbound connections, so no ports need to be opened.

## Installing Tailscale on the Server

From the Proxmox shell:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

I signed in through the link it gave me, then installed Tailscale on my laptop and phone using the same account.

## Reaching Every VM Through One Device

Rather than installing Tailscale on every VM, I set up the Proxmox server as a **subnet router**. It shares access to my whole home network with my other Tailscale devices.

```bash
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf
tailscale set --advertise-routes=LAN_SUBNET
```

Then I approved the route in the Tailscale admin console.

Now, when my laptop is connected to Tailscale, I can reach the Proxmox web interface and any VM on my home network from anywhere.

![Tailscale subnet route](images/02-tailscale-subnet-route.png)

## Verifying Nothing Is Exposed

To confirm my network wasn't exposed, I ran an external port scan against my public IP using GRC ShieldsUP!:

```
26 ports tested
 0 open
 0 closed
26 stealth
```

All tested ports came back as stealth, meaning my router doesn't respond to connection attempts from the internet. The only finding was that the router responds to ping, which shows the IP is in use but doesn't give any way in.

I also checked that:
- No port forwarding rules exist on the router
- UPnP is disabled, so devices can't open ports on their own

![ShieldsUP results](images/02-shieldsup-results.png)

## What I Learned

- Private IP addresses can't be reached from the internet directly. A home network usually only gets exposed through port forwarding or UPnP.
- A VPN like Tailscale gives full remote access without opening anything on the router.
- It's better to verify security with a scan than to assume it.
