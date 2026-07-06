# IP Addressing & Ports and Services — SOC Analyst Notes

## IP Addressing

### What an IP Address Actually Is

An IP address is a **logical address** assigned to a device on a network — logical meaning software-assigned and changeable, unlike a MAC address which is hardware-burned. It serves two purposes: **identifying a device** and **locating it** on a network so traffic can be routed to it.

---

### IPv4

**Structure:** 32 bits total, written as 4 groups of 8 bits (octets) separated by dots.

```
192  .  168  .   1  .  45
 ↑        ↑       ↑      ↑
8bits   8bits   8bits  8bits  = 32 bits total
```

Each octet ranges from 0–255 (8 bits = 2⁸ = 256 possible values).

#### Address Classes

| Class | Range | Default Mask | Typical use |
|---|---|---|---|
| **A** | 1.0.0.0 – 126.255.255.255 | /8 | Large networks, ISPs |
| **B** | 128.0.0.0 – 191.255.255.255 | /16 | Medium networks |
| **C** | 192.0.0.0 – 223.255.255.255 | /24 | Small networks, home/office |
| **D** | 224.0.0.0 – 239.255.255.255 | N/A | Multicast |
| **E** | 240.0.0.0 – 255.255.255.255 | N/A | Reserved/experimental |

#### Private vs Public IP Ranges — Critical for SOC

Reserved for internal use — **cannot be routed on the public internet:**

| Range | CIDR | Common use |
|---|---|---|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | Large enterprise networks |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | Medium networks |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | Home/small office (lab uses this) |

**SOC relevance:** private IP appearing as source on external firewall = **IP spoofing with private ranges** — immediate red flag.

#### Special Addresses — Recognize Instantly

| Address | Meaning | SOC relevance |
|---|---|---|
| **127.0.0.1** | Loopback — never leaves the machine | Should never appear in network traffic logs |
| **0.0.0.0** | Default route / "any address" | Used in firewall rules to mean "all sources" |
| **255.255.255.255** | Broadcast — sends to all devices on LAN | High volume = possible broadcast storm/attack |
| **169.254.x.x** | APIPA — DHCP failure fallback | Host has no valid network config — investigate why DHCP failed |

**Lab context:** during Kali setup, DHCP failed silently — if Kali had shown `169.254.x.x`, that would have been the immediate diagnostic signal. Recognizing APIPA addresses saves significant troubleshooting time.

---

### Subnetting and CIDR

**Subnetting** = dividing a large network into smaller sub-networks. CIDR (Classless Inter-Domain Routing) notation expresses this as a prefix length after a slash.

```
192.168.56.0/24
              ↑
              24 bits = network portion
              8 bits remaining = host addresses (254 usable hosts)
```

#### CIDR Quick Reference

| CIDR | Subnet mask | Usable hosts | Typical use |
|---|---|---|---|
| /8 | 255.0.0.0 | 16,777,214 | Large ISP/enterprise |
| /16 | 255.255.0.0 | 65,534 | Large campus network |
| /24 | 255.255.255.0 | 254 | Standard small network (lab) |
| /30 | 255.255.255.252 | 2 | Point-to-point links |
| /32 | 255.255.255.255 | 1 | Single host — used in firewall rules |

**Lab context:** Kali (`192.168.56.105`) and Metasploitable (`192.168.56.101`) are both on `192.168.56.0/24` — same /24 network, which is why they can communicate directly.

**Firewall rule context:** `/32` targets exactly one IP. `/24` targets all 254 hosts in that subnet. Always use `/32` in firewall rules when targeting a single specific host.

---

### IPv6

**Why it exists:** IPv4 only has ~4.3 billion addresses (2³²) — the internet ran out. IPv6 has 2¹²⁸ addresses, effectively unlimited.

**Structure:** 128 bits, written as 8 groups of 4 hex digits separated by colons:
```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```
Shortened (consecutive zero groups replaced with `::`):
```
2001:db8:85a3::8a2e:370:7334
```

#### IPv6 Addresses to Recognize

| Address | Meaning |
|---|---|
| `::1` | IPv6 loopback (equivalent to 127.0.0.1) |
| `fe80::/10` | Link-local — valid on local segment only, never routed |
| `ff00::/8` | Multicast range |

**Attacker abuse of IPv6:** many organizations enable IPv6 by default but don't monitor it — attackers exploit this blind spot by tunnelling traffic over IPv6 when IPv4 is monitored, or using IPv6 for C2 because security tools often have weaker IPv6 visibility.

---

### SOC IP Recognition Checklist — Run Through This Mentally on Every Log Entry

1. **Private?** (`10.x`, `172.16-31.x`, `192.168.x`) → internal traffic, shouldn't appear on external interfaces
2. **Loopback?** (`127.0.0.1` / `::1`) → should never appear in network logs
3. **APIPA?** (`169.254.x.x`) → DHCP failure on that host — investigate why
4. **Broadcast?** (`255.255.255.255`) → network-wide message
5. **Does source/dest make sense for traffic direction?** Private IP as source on public internet = red flag

---

## Self-Test — IP Addressing

### Q1: Firewall log shows inbound connection from source IP `192.168.1.45` on external firewall's public interface. Why suspicious?

**My answer:** Correct — `192.168.x.x` is a private range reserved for internal use only and cannot appear as a legitimate source on public internet traffic. Indicates **IP spoofing with private ranges.**

### Q2: Host shows IP `169.254.12.88`. What does this tell you and why is it a security concern?

**My answer:** DHCP failed, host assigned APIPA fallback. Correct.

**Correction/sharpening:** the real security concern isn't that the IP "leaks information" — it's that this is a signal to investigate *why* DHCP failed. Possible causes include a **DHCP starvation attack** (attacker exhausted the IP pool using all available leases) or a rogue DHCP server taking over. The APIPA address itself is the symptom; the cause is what matters.

### Q3: Writing iptables rule to block a single specific host — which CIDR?

**My answer:** `/32` — represents exactly one IP address, all 32 bits are the network portion, zero bits for hosts. `/24` would cover 254 hosts, which is too broad for targeting a single machine.

---

## Ports & Services

### What a Port Is

If an IP address is a building's street address, a **port is the specific door number.** One IP can run many services simultaneously — each listens on a different port so traffic knows which service to reach.

Port numbers range from **0 to 65535:**

| Range | Name | What's there |
|---|---|---|
| 0 – 1023 | Well-known ports | Standard services — HTTP, SSH, FTP etc. |
| 1024 – 49151 | Registered ports | Applications — MySQL (3306), RDP (3389) etc. |
| 49152 – 65535 | Dynamic/ephemeral ports | Temporarily assigned by OS for outbound connections |

---

### Well-Known Ports — Memorize These Cold

Every single one below appeared in the Metasploitable Nmap scan from Attack 1:

| Port | Protocol | Service | Risk profile |
|---|---|---|---|
| **20/21** | TCP | FTP | Plaintext, anonymous login abuse, file exfil |
| **22** | TCP | SSH | Brute force target, lateral movement |
| **23** | TCP | Telnet | Completely plaintext — should never exist on modern network |
| **25** | TCP | SMTP | Phishing, spam relay, BEC |
| **53** | UDP/TCP | DNS | Tunnelling, spoofing, DGA, cache poisoning |
| **67/68** | UDP | DHCP | Starvation attacks, rogue DHCP servers |
| **80** | TCP | HTTP | Plaintext web — credential theft, malicious downloads |
| **110** | TCP | POP3 | Email retrieval — plaintext credentials |
| **139/445** | TCP | SMB | Ransomware spread, lateral movement, EternalBlue |
| **143** | TCP | IMAP | Email access — plaintext or weakly encrypted |
| **443** | TCP | HTTPS | Encrypted web — C2 hiding, exfil over encrypted channel |
| **1433** | TCP | MSSQL | Database attack target |
| **3306** | TCP | MySQL | Database attack target (found in Nmap scan) |
| **3389** | TCP | RDP | Brute force, ransomware lateral movement — massive target |
| **4444** | TCP | Metasploit default | Reverse shell — if seen outbound, assume compromise |
| **5900** | TCP | VNC | Remote desktop — often poorly secured (found in Nmap scan) |
| **6667** | TCP | IRC | Botnet C2 historically, backdoors (found in Nmap scan) |
| **8080** | TCP | HTTP alternate | Web proxies, development servers |

---

### Ephemeral Ports — Why They Matter

When your machine makes an outbound connection, the OS assigns a **random high port** (49152–65535) as the source port:

```
Your machine: 192.168.1.5:52431 → Web server: 93.184.216.34:80
                              ↑
                    ephemeral port (OS-assigned, normal)
```

**SOC relevance:** high random source ports on outbound traffic = normal OS behavior. High random port as a **destination** from the same external IP consistently = suspicious (possible non-standard C2 channel).

---

### Ports as Attack Indicators

| What you see in logs | What it suggests |
|---|---|
| External IP repeatedly hitting port 22 | SSH brute force |
| Internal host connecting outbound on port 4444 | Metasploit reverse shell — likely compromised |
| Traffic on port 23 anywhere on a modern network | Telnet active — serious misconfiguration |
| Internal host hitting many ports sequentially | Port scan from inside — insider threat or compromised host doing lateral recon |
| External IPs hitting port 3389 | RDP brute force — common ransomware precursor |
| Outbound traffic on port 6667 | IRC-based botnet C2 |

---

### True Positive / False Positive / True Negative / False Negative

This terminology is used constantly in SOC work — know it cold:

| Term | Meaning | Example |
|---|---|---|
| **True Positive** | Alert fired, threat is real | SIEM flags malware — malware actually present |
| **False Positive** | Alert fired, no real threat | SIEM flags legitimate admin tool as malware |
| **True Negative** | No alert fired, no threat | Normal traffic, no alert — correct outcome |
| **False Negative** | No alert fired, threat IS real | Malware present but SIEM missed it — worst case |

**False negatives are the most dangerous** — something bad is happening and you have no idea.
**False positives are the most costly operationally** — alert fatigue causes analysts to miss real threats buried in noise.

---

## Self-Test — Ports & Services

### Q1: VNC on port 5900 found in Nmap scan. Colleague says "it's just remote desktop, it's fine." What follow-up questions determine real risk?

**My answer:** Initially said "is it properly configured and secured" — too vague.

**Full answer:** Is VNC authorized to run on this machine? Is it exposed to internet or internal only? Is it password protected and how strong? Is the traffic encrypted or plaintext? Who installed it and when — is it a new addition?

### Q2: Internal workstation making outbound connections to external IP on port 4444 every 30 seconds. What does this suggest and what are your first two Tier 1 actions?

**My answer:** Port 4444 = Metasploit reverse shell default, 30-second regularity = beaconing pattern. Likely compromised host.

**Terminology correction:** called this a "true negative" — **incorrect.** This is a **true positive** — alert fired, threat is real. True negative = no alert, no threat. Fix this terminology permanently.

**Two Tier 1 actions:** verify true positive (not a legitimate admin tool or scheduled task using that port), then escalate per playbook with evidence — do not unilaterally block without authorization.

### Q3: Why would an attacker run C2 on port 443 specifically?

**My answer:** Correct direction. **Precise reason:** port 443 is almost never blocked outbound by firewalls because blocking it would break HTTPS for the entire organization. Attacker uses it because it's **guaranteed to be open**, not because traffic automatically goes uninspected — NGFWs with SSL inspection do check it.

### Q4: Log shows `192.168.1.45:51892 → 8.8.8.8:53`. Suspicious or normal?

**My answer:** Normal — ephemeral source port (51892, OS-assigned for outbound), destination port 53 (DNS), 8.8.8.8 (Google's public DNS resolver). All four elements consistent with a legitimate DNS query. Correct on first attempt.
