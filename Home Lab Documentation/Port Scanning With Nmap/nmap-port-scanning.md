# Cybersecurity Homelab: Port Scanning with Nmap

A hands-on write-up from my isolated cybersecurity homelab. I used Nmap to scan a target server, then changed the server and scanned it again to prove that a change in the server's configuration shows up in the scan results.

> **Scope:** I only scanned machines I own, inside my own isolated lab network. Scanning systems you don't own or have permission to test is not acceptable.

This is the second write-up in the project. The first covers [packet analysis with Wireshark](README.md).

## Table of contents

1. [Lab environment](#lab-environment)
2. [Objective](#objective)
3. [Scan 1: baseline](#scan-1-baseline)
4. [Reading the results line by line](#reading-the-results-line-by-line)
5. [Changing the target: closing a door](#changing-the-target-closing-a-door)
6. [Scan 2: after stopping Apache](#scan-2-after-stopping-apache)
7. [Before and after](#before-and-after)
8. [How Nmap decided what it saw](#how-nmap-decided-what-it-saw)
9. [Security takeaways](#security-takeaways)
10. [Limitations and next steps](#limitations-and-next-steps)
11. [What I learned](#what-i-learned)

---

## Lab environment

| Machine | Role | Address | Notes |
|---|---|---|---|
| Kali Linux | Scanner | `10.0.0.10/24` | Runs Nmap as `root` |
| Debian | Target server | `10.0.0.20/24` | Running OpenSSH (22) and Apache2 (80) |

- Hypervisor: VirtualBox
- Network: both VMs share a VirtualBox **Internal Network** (`lab-net`), isolated from my real home network.
- Services on the target: `openssh-server` and `apache2`, installed earlier in the project so there was something to find.

## Objective

1. Use Nmap to discover which TCP ports are open on the target and what service is expected behind each.
2. Understand every line of Nmap's output, not just copy it.
3. Prove the scan reflects reality: stop a service on the target, rescan, and confirm the open port disappears. Then restore the service.

## Scan 1: baseline

**Command (on Kali):**

```bash
nmap 10.0.0.20
```

![Nmap scan of the target showing ports 22 and 80 open](images/07-nmap-baseline-scan.png)

**Result:**

```
Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-13 10:01 -0500
Nmap scan report for 10.0.0.20
Host is up (0.00068s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:9D:CE:27 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.70 seconds
```

## Reading the results line by line

| Output | What it means |
|---|---|
| `Starting Nmap 7.991 ... at 2026-11-13 10:01 -0500` | Nmap version and the scan start time. The `-0500` is the Kali VM's own timezone setting. |
| `Nmap scan report for 10.0.0.20` | The target being scanned. |
| `Host is up (0.00068s latency)` | The host answered, so it is online. The round trip was about 0.68 ms, which is expected for two VMs on the same virtual switch. |
| `Not shown: 998 closed tcp ports (reset)` | By default Nmap checks the **1,000 most common TCP ports**. 998 of them were closed. "Reset" means the target replied to each probe with a TCP RST, saying "nothing is listening here". |
| `22/tcp open ssh` | Port 22, TCP, is **open** and the expected service is SSH (remote login). |
| `80/tcp open http` | Port 80, TCP, is **open** and the expected service is HTTP (a web server). |
| `MAC Address: 08:00:27:9D:CE:27 (Oracle VirtualBox virtual NIC)` | The target's hardware address. It is shown because the target is on the same local network. It matches the MAC I saw in the ARP capture in the Wireshark write-up, and `08:00:27` is VirtualBox's vendor prefix. |
| `scanned in 0.70 seconds` | Total scan time. |

The numbers add up: 2 open ports plus 998 closed ports equals the 1,000 ports scanned.

## Changing the target: closing a door

To test whether the scan reflects the real state of the server, I stopped the web server on Debian:

```bash
systemctl stop apache2
```

![Debian terminal where the command systemctl stop apache2 was run as root](images/05-debian-stop-apache2.png)

The command prints nothing when it succeeds. Nothing is listening on port 80 anymore.

## Scan 2: after stopping Apache

I ran the same command again from Kali:

```bash
nmap 10.0.0.20
```

![Both Nmap scans in one terminal window; the second shows only port 22 open](images/08-nmap-after-stopping-apache.png)

**Result:**

```
Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-13 10:06 -0500
Nmap scan report for 10.0.0.20
Host is up (0.00064s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
MAC Address: 08:00:27:9D:CE:27 (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 0.79 seconds
```

Port 80 is gone from the open list, and the closed count went from **998 to 999**, which is the same port moving from "open" to "closed".

## Restoring the service

Once I had proven the point, I started Apache again so the lab returned to its working state:

```bash
systemctl start apache2
```

![Debian terminal where systemctl start apache2 was run as root](images/06-debian-start-apache2.png)

## Before and after

| | Scan 1 (Apache running) | Scan 2 (Apache stopped) |
|---|---|---|
| Time (Kali clock) | 10:01 | 10:06 |
| Open ports | 22 (ssh), 80 (http) | 22 (ssh) |
| Closed ports (reset) | 998 | 999 |
| Ports scanned | 1,000 | 1,000 |
| Scan time | 0.70 s | 0.79 s |

The only thing I changed on the target was the Apache service, and the only thing that changed in the scan was port 80. That is the evidence that the scan reflects the server's real state.

## How Nmap decided what it saw

- **Privilege and scan type.** I ran Nmap as `root`. In that case Nmap's default TCP method is a **SYN scan** (also called a half-open scan). It sends a SYN packet to each port. An open port answers with SYN-ACK, so Nmap knows a service is listening and resets the connection without completing the handshake. A closed port answers with RST. That RST is the `(reset)` in the output.
- **Host discovery.** Because the target is on the same local network, Nmap can discover the host through ARP, which is also why the MAC address is shown.
- **Closed versus filtered.** After Apache stopped, port 80 became **closed**, not "filtered". Closed means the target actively said "nothing here". Filtered would mean something, such as a firewall, was silently dropping the probes so Nmap couldn't tell. The target has no firewall in front of it at this stage of the lab, so every port gives a clear answer. That will change when I add a firewall later in the project.

## Security takeaways

- **Every open port is attack surface.** Each listening service is something an attacker can probe and try to exploit or log into. Defenders reduce risk by closing or removing services they don't need.
- **Scanning is how both sides see the network.** Attackers scan to find a way in. Defenders scan their own systems to check that only the intended ports are open, and to catch services that were left running by mistake.
- **Stopping a service really does remove the exposure.** The second scan is the proof. A change should always be verified, not assumed.
- **SSH on port 22 stays exposed.** Remote login services are common targets for password guessing, so a real server would harden it (key-based login, no root login, restricting who can connect).

## Limitations and next steps

- **Service names are guesses.** In this default scan, `ssh` and `http` come from the standard port-to-service table, not from asking the service. A different program can run on any port. Nmap's `-sV` option probes each open port to identify the actual software and version, and that is what lets a defender check whether it is out of date.
- **Only the top 1,000 ports were checked.** A service on an unusual port would be missed. The `-p-` option scans all 65,535 ports.
- **TCP only.** UDP services are not covered by this scan.
- **Next:** repeat the scan with `-sV`, `-p-` and a ping sweep (`-sn 10.0.0.0/24`), capture a scan in Wireshark to see the flood of SYN and RST packets, and rescan after the firewall (OPNsense) is added to see ports appear as filtered.

## What I learned

- What each part of an Nmap report means, and how the open and closed counts add up to the number of ports scanned
- The difference between **open**, **closed** and **filtered** ports
- Why Nmap's default scan type depends on whether it runs as root
- How to verify a change: scan, change one thing, scan again, and compare
- That default scan results show expected services by port number, and that `-sV` is needed to confirm what is really running

## Tools used

VirtualBox · Kali Linux · Debian · Nmap · systemctl · Apache2 · OpenSSH

---

*Lab built and documented on 13 November 2026.*
