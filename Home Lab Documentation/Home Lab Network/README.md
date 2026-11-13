# Homelab Network: Kali ↔ Debian Target

A small isolated virtual network built in VirtualBox for cybersecurity practice. A Kali Linux attacker machine and a Debian target machine share a private `10.0.0.0/24` lab network, so I can build, test and break things safely.

## Lab Overview

| VM | Role | OS | Lab interface | Lab IP | NAT interface | NAT IP |
|----|------|----|---------------|--------|---------------|--------|
| Kali | Attacker | Kali Linux 2026.3 | `eth1` | `10.0.0.10/24` | `eth0` | `10.0.2.15/24` (DHCP) |
| Debian | Target | Debian | `enp0s8` | `10.0.0.20/24` | `enp0s3` | `10.0.2.15/24` (DHCP) |

Each VM has two network adapters:

- **Adapter 1 (NAT):** Internet access for updates and package installs. Each VM gets `10.0.2.15` from VirtualBox's built-in DHCP, but the two NATs are separate, so they cannot reach each other this way.
- **Adapter 2 (lab network):** The private `10.0.0.0/24` network where the two machines talk to each other.
  - Network mode: `TODO: Internal Network / Host-only (name: ____)`

## Topology

```
        +------------------+                 +------------------+
        |   Kali (attacker)|                 |  Debian (target) |
        |                  |                 |                  |
        | eth0  10.0.2.15  |--- NAT          | enp0s3 10.0.2.15 |--- NAT
        | eth1  10.0.0.10  |=== 10.0.0.0/24 ===| enp0s8 10.0.0.20 |
        +------------------+    lab network   +------------------+
```

## Build Steps

### 1. Add a second network adapter to each VM
In VirtualBox: VM Settings → Network → Adapter 2 → enable, and attach both VMs to the same lab network (`TODO: mode and network name`).

### 2. Confirm the new interface
On each VM:

```bash
ip a
```

The second interface (`eth1` on Kali, `enp0s8` on Debian) shows `UP` but has no IPv4 address yet.

### 3. Assign static IPs

Kali:
```bash
sudo ip addr add 10.0.0.10/24 dev eth1
```

Debian:
```bash
sudo ip addr add 10.0.0.20/24 dev enp0s8
```

> The `dev <interface>` part is required. Without it, `ip` fails with: `Not enough information: "dev" argument is required.`

### 4. Verify connectivity
From Kali:

```bash
ping -c 4 10.0.0.20
```

Result: 4 packets sent, 4 received, 0% packet loss, ~0.5 ms average latency.

## Troubleshooting Notes

- **`Destination Host Unreachable` on the first ping:** The ping ran before Debian had its `10.0.0.20` address, so nothing on the lab network answered. After adding the address on Debian, the ping succeeded. Always assign IPs on both ends before testing.
- **Addresses disappear after a reboot:** `ip addr add` is not persistent. Make the addresses permanent (for example in `/etc/network/interfaces` on Debian, or a NetworkManager connection) if the lab should survive restarts.

## Screenshots

| Description | File |
|-------------|------|
| Debian: assigning `10.0.0.20` and checking `ip a` | `screenshots/debian-ip-config.png` |
| Kali: assigning `10.0.0.10` and checking `ip a` | `screenshots/kali-ip-config.png` |
| Kali: failed ping, then successful ping to Debian | `screenshots/kali-ping-debian.png` |

## What I Learned

- How VirtualBox NAT differs from a private lab network
- Reading `ip a` output: interface names, MAC addresses, IPv4 and IPv6 addresses
- Assigning static IPs and testing reachability with `ping`
- Why a failed ping can come from the other end not being configured yet

## Next Steps

- [ ] Make the IP configuration persistent
- [ ] Run an `nmap` scan from Kali against the Debian target
- [ ] Add a service to Debian (SSH, web server) and enumerate it from Kali
- [ ] Capture and analyse the traffic with Wireshark
