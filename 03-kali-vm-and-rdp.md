# 3. Kali Linux VM and Remote Desktop

Next I set up a Kali Linux VM to use as my penetration testing workstation, with a full desktop I can access remotely.

**End result:** Kali running in Proxmox, accessed through Remote Desktop over Tailscale, with RDP blocked on every other network.

## Getting the ISO

Proxmox can download ISO files directly from a URL, so I didn't need to download it to my PC first. One thing I ran into: the link has to point directly to the `.iso` file. My first attempt used the link from the website's download page, and Proxmox rejected it with a "wrong file extension" error.

## VM Settings

| Setting | Value | Reason |
|---|---|---|
| CPU | 2 cores, type `host` | Passes the real CPU features to the VM for better performance |
| Memory | 4 GB | Enough for the desktop and tools |
| Disk | 64 GB | Room for Kali and its tools |
| Network | VirtIO on `vmbr0` | Faster virtual network card |
| QEMU Guest Agent | Enabled | Lets Proxmox see the VM's IP and shut it down cleanly |

![Kali VM hardware](images/03-kali-hardware.png)

I installed Kali with the default desktop and tools, then updated it and installed the guest agent:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

I also set the VM to start automatically when the server boots.

## Setting Up Remote Desktop

The browser console in Proxmox works but is slow, so I set up RDP with xrdp:

```bash
sudo apt install -y xrdp xorgxrdp
sudo systemctl enable --now xrdp
echo "xfce4-session" > ~/.xsession
```

Then I connected from Windows using Remote Desktop Connection.

## Problem: Session Crashed After Login

The xrdp login screen loaded, but as soon as I entered my password, the session crashed.

I found two causes:

1. **Missing permission.** The xrdp service needs to be in the `ssl-cert` group to read its certificate.
   ```bash
   sudo adduser xrdp ssl-cert
   sudo systemctl restart xrdp
   ```
2. **Session conflict.** I was still logged in to Kali through the Proxmox console. xrdp can't start a second desktop session for a user who is already logged in locally. I rebooted and only logged in through RDP.

After both fixes, RDP worked. I now log out of Kali when I'm done instead of just closing the window, so the session doesn't stay open.

![Kali over Remote Desktop](images/03-kali-rdp-desktop.png)

## Restricting RDP to the VPN

I installed Tailscale directly on Kali as well, which gives it an address that doesn't change when the router reassigns IPs.

By default, xrdp accepts connections on every network the VM is connected to, including my home network. To reduce that exposure, I used the UFW firewall to only allow RDP through Tailscale:

```bash
sudo apt install -y ufw
sudo ufw allow in on tailscale0 to any port 3389
sudo ufw deny 3389
sudo ufw enable
```

Now RDP only works over the VPN.

## Result

- Kali VM running with remote desktop access
- Fixed the xrdp crash on login
- RDP restricted to the Tailscale VPN only
- VM starts automatically with the server
