# SSH Homelab Access Setup

**Date:** 13 November 2026

## Goal

Set up SSH (Secure Shell) so the Windows PC hosting the homelab (Debian VM + Kali in VirtualBox) can be accessed remotely from a laptop — the first step toward running lab exercises from the laptop without sitting at the main PC.

## Steps Taken

1. **Installed OpenSSH Server on the Windows PC**
   Enabled via Settings > Apps > Optional Features (or `Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0` in an elevated PowerShell).

2. **Started the SSH service and set it to auto-start**
   ```
   Start-Service sshd
   Set-Service -Name sshd -StartupType Automatic
   ```
   This ensures `sshd` starts automatically on every boot.

3. **Verified the Windows Firewall rule**
   `Get-NetFirewallRule -Name *ssh*` confirmed an existing inbound rule (`OpenSSH-Server-In-TCP`), enabled, set to Allow, scoped to the **Private** network profile.

4. **Found the PC's IP address**
   Used `ipconfig` to get the PC's local IPv4 address on the home network.

5. **Identified the correct login username**
   `whoami` returned `<computer-name>\<username>` — the part before the backslash is the computer name, the part after is the actual account username to use for SSH.

## Troubleshooting: Connection Timed Out

The first SSH attempt from the laptop failed with a connection timeout. Diagnosed step by step:

- **Ping test** from laptop to PC also timed out (100% packet loss) — but this turned out to be a red herring, since Windows commonly blocks ICMP (ping) separately from other traffic.
- Confirmed laptop and PC were on the **same subnet** (both `192.168.1.x`), ruling out a network-separation issue.
- Confirmed the PC's Windows network profile was correctly set to **Private** (`Get-NetConnectionProfile`), so the firewall rule was active.
- Confirmed `sshd` was listening and working locally on the PC itself:
  ```
  Test-NetConnection -ComputerName localhost -Port 22
  ```
  Returned `TcpTestSucceeded : True`.
- **Key test** — from the laptop, tested port 22 specifically (rather than ping):
  ```
  Test-NetConnection -ComputerName <pc-local-ip> -Port 22
  ```
  Returned `TcpTestSucceeded : True` — confirming port 22 was actually reachable all along. The earlier ping failure was unrelated (ICMP-specific), not a real blocker.

## Result

Successfully connected from the laptop to the Windows PC:
```
ssh <username>@<pc-local-ip>
```
- Accepted the one-time host key fingerprint prompt (`yes`).
- Entered the Windows account password (not the sign-in PIN — PINs are device-specific and don't work over SSH).
- Landed at a working Windows command prompt, confirmed via the "Microsoft Corp... all rights reserved" banner.

## Basic SSH Usage Learned

- SSH gives a **command-line only** session — not a view of the remote desktop/screen.
- Commands run as if typed directly on the PC (`dir`, `cd`, `whoami`, `ipconfig`, etc.).
- `exit` closes the session cleanly and returns to the laptop's own prompt.
- Reconnecting later just reuses the same command (`ssh <username>@<pc-local-ip>`); the fingerprint prompt only appears the first time.

## Next Steps

- **SSH "hopping" into the VMs** — from inside the PC's SSH session, SSH again into Debian/Kali to reach the actual lab machines. Requires:
  - Setting the VMs' VirtualBox network adapters to **Bridged** (so they get their own LAN-visible IP) or **Host-only** (reachable only via the PC).
  - Installing/enabling OpenSSH server inside each VM (`sudo apt install openssh-server -y && sudo systemctl enable --now ssh`).
- **Remote access from outside the home network** — recommended approach is **Tailscale** (a private virtual network between devices, no exposed ports) rather than traditional port forwarding, which exposes port 22 directly to internet-wide scanning and attack.
- **Packet capture workflow** — Wireshark's GUI doesn't work over plain SSH, but its command-line counterpart `tshark` does. Planned workflow: capture with `tshark` over SSH on the remote VM, pull the `.pcap` file back with `scp`, and analyze it in the full Wireshark GUI locally.
