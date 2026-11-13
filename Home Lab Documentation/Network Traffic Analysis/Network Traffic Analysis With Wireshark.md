# Cybersecurity Homelab: Network Traffic Analysis with Wireshark

A hands-on write-up from my isolated cybersecurity homelab. I built a small virtual network, generated traffic between an attacker machine and a target server, and captured it with Wireshark to see how ICMP, ARP and TCP/HTTP behave on the wire.

> **Scope:** Everything here was done inside my own isolated lab network. No real or third-party systems were involved.

## Table of contents

1. [Lab environment](#lab-environment)
2. [Objective](#objective)
3. [Capture setup](#capture-setup)
4. [Exercise 1: ICMP (ping)](#exercise-1-icmp-ping)
5. [Exercise 2: ARP (finding a neighbour's MAC address)](#exercise-2-arp-finding-a-neighbours-mac-address)
6. [Exercise 3: TCP three-way handshake and HTTP](#exercise-3-tcp-three-way-handshake-and-http)
7. [Background traffic](#background-traffic)
8. [Security takeaways](#security-takeaways)
9. [What I learned](#what-i-learned)

<!-- Nmap section goes here -->

---

## Lab environment

| Machine | Role | Address | Notes |
|---|---|---|---|
| Kali Linux | Attacker / analysis machine | `10.0.0.10/24` on `eth1` | Runs Wireshark and the test commands |
| Debian | Target server | `10.0.0.20/24` | Running OpenSSH (22) and Apache2 (80) |

- Hypervisor: VirtualBox
- Network: both VMs share a VirtualBox **Internal Network** called `lab-net`. It is a sealed virtual switch, so only VMs attached to it can see each other. Not even the host PC can.
- Each VM also has a separate NAT adapter used only for downloads and updates. I never used Bridged mode, so the lab can't touch my real home network.
- Addresses are static, because there is no DHCP server on `lab-net`.

## Objective

Instead of only trusting that "ping works" or "the web page loads", I wanted to see the individual packets behind three everyday network actions and be able to explain each one:

1. A ping (ICMP echo request and reply)
2. How a host discovers a neighbour's MAC address (ARP)
3. How a TCP connection is established, used and closed, using an HTTP request to the web server

## Capture setup

- Tool: Wireshark on Kali
- Interface: **`eth1`**, the adapter on `lab-net`. (`eth0` is the NAT adapter and would show none of the lab traffic.)
- Traffic generated from Kali's terminal:
  - `ping -c 4 10.0.0.20`
  - `curl http://10.0.0.20`
- Display filters used: `icmp`, `arp`, `tcp.port == 80`

### The whole capture, unfiltered

![Wireshark capture on eth1 with no filter, alongside a ping in the terminal](images/01-capture-overview.png)

With no filter, the capture mixes several protocols. The coloured rows show:

- **ICMP** ping requests and replies between `10.0.0.10` and `10.0.0.20`
- **ARP** address lookups
- **ICMPv6 and MDNS**: background chatter from the operating systems (see [Background traffic](#background-traffic))

Display filters let me pull one conversation out of this stream at a time.

---

## Exercise 1: ICMP (ping)

**Command:** `ping -c 4 10.0.0.20` **Filter:** `icmp`

![Wireshark filtered to ICMP showing four echo requests and four echo replies](images/02-icmp-ping.png)

### What I observed

- `4 packets transmitted, 4 received, 0% packet loss`
- Wireshark shows **8 ICMP packets**: 4 *Echo (ping) request* from `10.0.0.10` to `10.0.0.20`, each followed by an *Echo (ping) reply* going back.
- All eight share the same ICMP identifier (`id=0x4496`), and the sequence numbers run `seq=1` to `seq=4`. That is how ping matches each reply to its request.
- Each packet is **98 bytes** on the wire: 14 bytes Ethernet header + 20 bytes IPv4 header + 8 bytes ICMP header + 56 bytes of data. That matches the `56(84) bytes of data` line from `ping` (56 data + 8 ICMP + 20 IP = 84).
- Replies show `ttl=64`, which is the default starting TTL for Linux, so the target is one hop away with no router in between.
- The packet detail pane shows the layers stacked on top of each other: **Ethernet II, then IPv4, then ICMP**.

### Why the first ping was slower

The terminal shows `time=1.34 ms` for the first reply and about `0.6 ms` for the rest. The capture explains why. In the ARP capture below, Kali's ARP request goes out at `16.299559` and the first echo reply comes back at `16.300888`. That is about 1.33 ms, matching what `ping` reported. For the first packet, `ping`'s timer includes the time spent on the **ARP lookup**. By packets 2 to 4 the MAC address was cached, so only the true round trip (about 0.6 ms) was left.

---

## Exercise 2: ARP (finding a neighbour's MAC address)

**Filter:** `arp`

![Wireshark filtered to ARP showing a broadcast request and a reply](images/03-arp-resolution.png)

IP addresses decide *which host* a packet is for, but on a local network the frame is actually delivered using **MAC addresses**. Before Kali can send anything to `10.0.0.20`, it has to learn Debian's MAC address. ARP does that.

I first cleared Kali's neighbour (ARP) cache with `sudo ip neigh flush all`, so the lookup would have to happen again and I could capture it.

### What I observed

| Frame | Source | Destination | Info |
|---|---|---|---|
| 2 | Kali (`08:00:27:9b:5e:a1`) | **Broadcast** (`ff:ff:ff:ff:ff:ff`) | `Who has 10.0.0.20? Tell 10.0.0.10` |
| 3 | Debian (`08:00:27:9d:ce:27`) | Kali | `10.0.0.20 is at 08:00:27:9d:ce:27` |

- The **request is a broadcast**: Kali shouts to every device on the segment, because it doesn't yet know who owns that IP.
- The **reply is unicast**: Debian answers Kali directly with its MAC address.
- Frame 2 is 42 bytes on the wire: a 14-byte Ethernet header plus a 28-byte ARP message.
- The `08:00:27` prefix is VirtualBox's vendor ID (Wireshark labels it `PCSSystemtec`).
- Later in the capture there are more ARP exchanges, including one started by Debian asking `Who has 10.0.0.10? Tell 10.0.0.20`. Hosts refresh their cached entries from time to time, so ARP traffic keeps appearing on a quiet network.

---

## Exercise 3: TCP three-way handshake and HTTP

**Command:** `curl http://10.0.0.20` **Filter:** `tcp.port == 80`

![Wireshark filtered to port 80 showing SYN, SYN-ACK, ACK, an HTTP GET, a 200 OK, and the connection teardown](images/04-tcp-handshake-http.png)

`curl` fetched the Apache default page from the Debian server. Terminal output shows the HTML, including `<title>Apache2 Debian Default Page: It works</title>`. Wireshark captured the whole connection in **12 packets**.

### The connection, step by step

| Frames | Packet | What it means |
|---|---|---|
| 38 | `[SYN]` Kali → Debian | "Can we talk?" Kali uses a random source port (`32958`) and targets port `80`. |
| 39 | `[SYN, ACK]` Debian → Kali | "Yes. Can you hear me?" |
| 40 | `[ACK]` Kali → Debian | "Yes, let's go." The **three-way handshake** is complete. |
| 41 | `HTTP GET / HTTP/1.1` | Kali asks for the home page. |
| 42 | `[ACK]` | Debian confirms it received the request. |
| 43 | `[PSH, ACK]` (7306 bytes) | Debian starts sending the page data. |
| 44 | `[ACK]` | Kali acknowledges the data. |
| 45 | `HTTP/1.1 200 OK (text/html)` | The HTTP response. `200 OK` means success. |
| 46 | `[ACK]` | Kali acknowledges the rest of the page. |
| 47 | `[FIN, ACK]` Kali → Debian | Kali starts closing the connection. |
| 48 | `[FIN, ACK]` Debian → Kali | Debian closes its side too. |
| 49 | `[ACK]` | Final acknowledgement. The connection is fully closed. |

### Details worth noting

- **Handshake speed:** the handshake (frames 38 to 40) took under 1 ms, and the whole exchange, from first SYN to final ACK, finished in about 6 ms.
- **Sequence and acknowledgement numbers:** the ACKs count bytes. For example, after Kali's 73-byte GET request, Debian's acknowledgement number is `74`. By the end, the acknowledgement number is about `10959`, so roughly 11 KB of page was delivered.
- **Large frame size:** the 7306-byte segment is bigger than a normal Ethernet frame (about 1500 bytes). This is most likely because the virtual network adapter hands the capture a larger segment than would appear on a physical wire (segmentation offloading), rather than the page really travelling as one giant frame.
- **Plain text:** because this is HTTP and not HTTPS, the request and the HTML page were readable in the capture.

---

## Background traffic

Even before I generated any traffic, the unfiltered capture had other protocols in it:

- **ICMPv6 Router Solicitation** (`ff02::2`): the machines' IPv6 stacks looking for an IPv6 router.
- **MDNS** (`224.0.0.251` and `ff02::fb`): multicast DNS service-discovery queries (for example `_ipp._tcp.local`, which looks for network printers).

No one asked for these. They come from the operating systems' own services. This matters for analysis: a real capture is noisy, and being able to filter and identify what is normal is a core skill before you can spot what isn't.

## Security takeaways

- **Cleartext protocols can be read by anyone on the path.** The whole HTTP request and the page content were visible. This is why HTTPS and SSH exist. Anyone able to capture traffic on a shared segment could see this data.
- **ARP has no authentication.** Any host can answer an ARP request, and hosts typically accept the answer. This is why ARP spoofing and man-in-the-middle attacks are possible on a local network, and why defenders watch for unusual ARP activity.
- **Packet captures are evidence.** Analysts use captures to confirm what actually happened on a network, for example to identify a scan, a suspicious connection or data leaving the network.
- **A closed door and a silent door look different on the wire.** Handshake traffic (SYN, then SYN-ACK or RST) is what port scanners rely on. That is covered in the Nmap section of this project.

## What I learned

- How a ping works at the packet level, and how request and reply are matched using the ICMP identifier and sequence number
- Why ARP is needed, why its request is a broadcast and its reply is unicast, and how ARP lookup time shows up in a ping's first round trip
- How to read a TCP conversation from handshake to teardown, including SYN, SYN-ACK, ACK, PSH and FIN flags
- How to use Wireshark display filters (`icmp`, `arp`, `tcp.port == 80`) to isolate one conversation from a noisy capture
- Why the right capture interface matters: `eth1` for lab traffic, not the NAT adapter
- How to think about each packet in layers: Ethernet (MAC addresses), IP (IP addresses), then ICMP or TCP, then the application protocol (HTTP)

## Tools used

VirtualBox · Kali Linux · Debian · Wireshark · ping · curl · Apache2

---

*Lab built and documented on 13 November 2026.*
