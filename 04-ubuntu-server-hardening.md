# 4. Ubuntu Server Setup and Hardening

I added an Ubuntu Server VM to practise managing a Linux server the way it's done in real environments: no desktop, managed remotely over SSH. After installing it, I hardened it to a secure baseline.

**End result:** Ubuntu Server with SSH key authentication only, root login disabled, a firewall, and automatic security updates.

## Why Ubuntu Server Instead of Desktop

- Real servers don't run a desktop and are managed over SSH
- It uses about 1 GB of RAM instead of 4 GB or more, leaving room for other VMs
- It's a realistic system to defend, and to attack later in the lab

## VM Settings

| Setting | Value |
|---|---|
| CPU | 2 cores, type `host` |
| Memory | 2 GB |
| Disk | 32 GB |
| Network | VirtIO on `vmbr0` |
| QEMU Guest Agent | Enabled |

During the install, I chose the standard Ubuntu Server option and selected **Install OpenSSH server**. After it finished, I updated the system and installed the guest agent:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y qemu-guest-agent
sudo systemctl start qemu-guest-agent
```

I connect to it from my PC with:

```bash
ssh <user>@VM_IP
```

This works at home and remotely through the Tailscale subnet route.

## Hardening

### SSH Key Authentication

I generated an Ed25519 key on my PC, protected with a passphrase:

```powershell
ssh-keygen -t ed25519
```

Then I copied the public key to the server. Windows doesn't have `ssh-copy-id`, so I sent it over SSH:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <user>@VM_IP "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"
```

### Disabling Password and Root Login

I created `/etc/ssh/sshd_config.d/00-hardening.conf`:

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

I named the file starting with `00-` because SSH uses the first value it reads for each setting, and these files load in alphabetical order. That makes sure these settings take priority.

```bash
sudo systemctl restart ssh
sudo sshd -T | grep -E "passwordauthentication|permitrootlogin"
```

Before closing my session, I tested the new login in a second terminal so I wouldn't lock myself out. If I ever do get locked out, I can still use the Proxmox console.

### Firewall

I enabled UFW to block all incoming traffic except SSH:

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

### Automatic Security Updates

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### Snapshot

Once everything was set up, I took a Proxmox snapshot called `hardened` so I can roll back to a clean, secure state after experimenting.

![Hardening verification](images/04-ubuntu-hardening.png)

## Result

| Security Control | Status |
|---|---|
| SSH key authentication | Enabled |
| Password login | Disabled |
| Root login over SSH | Disabled |
| Firewall (UFW) | Enabled, SSH only |
| Automatic security updates | Enabled |
| Snapshot of clean state | Taken |

These changes protect against some of the most common ways Linux servers get compromised: guessed passwords, direct root access, unnecessary open ports, and missing updates.
