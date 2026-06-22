# Packet Analysis with Wireshark — SOC Analyst Notes

## What it does

Wireshark captures and displays every packet flowing through a network interface, broken down layer by layer. It's the tool for actually *seeing* handshakes, attacks, and protocols in real packets rather than just theory.

## Attacker use

Attackers use packet capture (Wireshark, tcpdump) for reconnaissance — sniffing unencrypted traffic to steal credentials, session tokens, or sensitive data. This is exactly why unencrypted HTTP and Telnet are dangerous: anything in plaintext is fully visible to anyone capturing packets on that network.

## Defender use

SOC analysts use Wireshark to investigate incidents after the fact — pulling a PCAP (packet capture file) from a firewall, IDS, or network tap, and manually inspecting it to confirm what happened during an attack.

## What it looks like

Each row = one packet. Columns: time, source IP, destination IP, protocol, info. Every packet expands to show all 7 OSI layers stacked — click into Ethernet (L2), IP (L3), TCP (L4), application data (L7) for any single packet.

---

## Useful Filters

| Filter | What it shows |
|---|---|
| `tcp.flags.syn==1 and tcp.flags.ack==0` | SYN packets with no ACK — useful for spotting SYN floods |
| `http` | All HTTP traffic |
| `dns` | All DNS queries/responses |
| `ip.addr==192.168.1.10` | All traffic to/from a specific IP |
| `arp` | All ARP traffic — useful for spotting ARP spoofing |
| `tcp.port==445` | SMB traffic — common ransomware/lateral movement port |

---

## Normal vs Suspicious Patterns

**Normal TCP handshake:**
```
192.168.1.5 → 10.0.0.1   [SYN]
10.0.0.1 → 192.168.1.5   [SYN, ACK]
192.168.1.5 → 10.0.0.1   [ACK]
```
Clean, 3 packets, milliseconds.

**SYN flood:**
One IP (or many spoofed IPs) spamming SYNs to one destination, no ACKs ever returned. High volume from one/few sources to one target is the actual red flag — not retransmissions alone.

**ARP spoofing:**
Repeated ARP replies claiming the same IP belongs to different MAC addresses. Wireshark sometimes flags this directly: "Duplicate IP address detected."

**DNS tunnelling:**
Filter by `dns` — abnormally long, random-looking subdomains queried repeatedly to the same parent domain.

---

## Lab Exercise — Live Capture

1. Installed Wireshark, selected active interface (Wi-Fi), started capture
2. Visited `http://neverssl.com` (deliberately unencrypted HTTP test site)
3. Filtered by `http`, found the GET request, right-clicked → Follow → HTTP Stream
4. Filtered by `tcp.flags.syn==1` to inspect handshake packets

### Findings

**1. Plaintext HTTP capture confirmed:**
Full request/response visible in clear text — `GET / HTTP/1.1`, `Host: neverssl.com`, complete browser User-Agent string, and the full server response (`HTTP/1.1 200 OK`, Apache server headers, HTML body). Proves nothing in HTTP is encrypted — a login form on an HTTP site would expose credentials exactly the same way.

**2. SYN filter results — retransmissions, not an attack:**
Captured several `[TCP Retransmission]` SYN packets to different destination IPs/ports (mostly background browser traffic) that never received a SYN-ACK back, alongside one connection (port 80, to the actual NeverSSL server) that completed normally with a SYN-ACK response.

**Why retransmissions happen without a SYN-ACK reply:**
- Destination IP/port not actually listening
- Firewall silently dropping the SYN (drop vs reject — drop gives no response at all)
- Packet lost in transit
- High latency — OS retry timer fires before the real SYN-ACK arrives

**Key correction from this exercise:** I initially assumed a "verification" step happens before the server replies with SYN-ACK. That's wrong — **SYN-ACK is the server's direct, immediate reply to the SYN.** No intermediate verification step exists in the handshake itself.

**Key lesson — don't over-call retransmissions as an attack:**
A handful of retransmissions across multiple different destinations (normal browser noise — slow servers, distant geography, brief congestion) looks completely different from a real SYN flood, which shows **high volume from one or few sources converging on a single destination with zero completions ever**. Volume and pattern matter more than the mere presence of retransmissions.
