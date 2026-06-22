# OSI Model — SOC Analyst Notes

## Why it matters for SOC work

When an attack happens, it happens at a specific layer. Knowing which layer tells you what tool catches it, what logs to check, and how to contain it. A DDoS hits Layer 4. Phishing hits Layer 7. ARP spoofing hits Layer 2. Without this model, you're guessing blind.

---

## The 7 Layers — Top to Bottom

### Layer 7 — Application

| | |
|---|---|
| **What it does** | The layer humans interact with. HTTP, HTTPS, DNS, FTP, SMTP, SSH all live here. |
| **Attacker use** | Phishing emails (SMTP), malicious websites (HTTP/S), DNS tunnelling for data exfiltration, SQL injection through web apps |
| **Defender detects** | Web proxy logs, DNS query logs, email gateway alerts, WAF blocks |
| **In logs** | `GET /malicious-payload.exe HTTP/1.1`, unusual DNS queries to random domains |

### Layer 6 — Presentation

| | |
|---|---|
| **What it does** | Translates data into a format the application understands. Handles encryption (SSL/TLS), compression, encoding |
| **Attacker use** | SSL stripping (downgrading HTTPS to HTTP), hiding malware inside Base64-encoded payloads |
| **Defender detects** | TLS inspection, detecting unencrypted traffic on port 443, spotting Base64 encoded commands in logs |
| **In logs** | PowerShell logs showing `[System.Convert]::FromBase64String(...)` |

### Layer 5 — Session

| | |
|---|---|
| **What it does** | Opens, manages, and closes communication sessions between two machines |
| **Attacker use** | Session hijacking — stealing an active authenticated session token to impersonate a user |
| **Defender detects** | Same session ID used from two different IPs, abnormally long sessions |
| **In logs** | Auth logs showing same session token from different geographic IPs |

### Layer 4 — Transport

| | |
|---|---|
| **What it does** | End-to-end communication. TCP (reliable, connection-based) and UDP (fast, connectionless). Port numbers live here |
| **Attacker use** | Port scanning, SYN flood DDoS, UDP flood |
| **Defender detects** | IDS/IPS signatures, firewall logs showing SYN without ACK, Nmap scan patterns |
| **In logs** | `SYN from 192.168.1.45 → port 22, 23, 80, 443, 445, 3389` in rapid succession = port scan |

### Layer 3 — Network

| | |
|---|---|
| **What it does** | Logical addressing (IP addresses) and routing |
| **Attacker use** | IP spoofing, ICMP tunnelling, routing table poisoning |
| **Defender detects** | Firewall/router logs, anomalous ICMP volume, spoofed IPs failing reverse lookup |
| **In logs** | `ICMP echo request from 10.0.0.5 — 4,000 packets in 60 seconds` |

### Layer 2 — Data Link

| | |
|---|---|
| **What it does** | Physical addressing (MAC addresses), transfers data between devices on the same local network |
| **Attacker use** | ARP spoofing — fake ARP replies associating attacker's MAC with a legitimate IP, enabling MITM |
| **Defender detects** | ARP monitoring, dynamic ARP inspection on switches, duplicate MAC-IP mappings |
| **In logs** | Same IP resolving to two different MACs in the ARP table |

### Layer 1 — Physical

| | |
|---|---|
| **What it does** | Actual physical transmission of raw bits — cables, switches, wireless signals, hardware |
| **Attacker use** | Rogue USB (rubber ducky), cable tapping, evil twin WiFi |
| **Defender detects** | Physical security controls, rogue AP detection, 802.1X network access control |
| **In logs** | New unrecognised MAC on the switch, rogue SSID matching corporate SSID |

---

## Attack-to-Layer Cheat Sheet

| Attack | Layer |
|---|---|
| Phishing / malicious URL | 7 |
| SSL stripping / Base64 malware | 6 |
| Session hijacking | 5 |
| Port scan / SYN flood / DDoS | 4 |
| IP spoofing / ICMP tunnel | 3 |
| ARP spoofing / MITM | 2 |
| Rogue USB / evil twin WiFi | 1 |

## TCP/IP Model Mapping

| TCP/IP Layer | OSI Equivalent |
|---|---|
| Application | Layers 5, 6, 7 |
| Transport | Layer 4 |
| Internet | Layer 3 |
| Network Access | Layers 1, 2 |

---

## Self-Test — Mistakes I Made (and corrected)

**Q1: Thousands of SYN packets, no completed handshake — which layer/attack?**
- My first answer: Session layer (wrong)
- Correct answer: **Layer 4 (Transport)** — SYN flood attacks the TCP handshake mechanism, which lives at Transport, not Session. Session layer manages already-established sessions.

**Q2: ARP table shows two MACs resolving to the same IP — which layer/attack?**
- My answer: **Layer 2 (Data Link)** — ARP spoofing. Correct on first try.
- Goal: attacker poisons the ARP cache so traffic meant for the real IP routes through them first (MITM setup).

**Q3: PowerShell running Base64-encoded commands — which layer, and why?**
- My first answer: Session layer (Layer 5) — wrong
- Correct answer: **Layer 6 (Presentation)** — encoding/obfuscation lives here. Attackers Base64-encode payloads to **obfuscate** them — bypass signature-based detection without actually encrypting the data. A tool scanning for `malware.exe` won't flag `bWFsd...`.

**Key lesson:** Layer 5 (Session) vs Layer 6 (Presentation) is my weak point. Session = managing active connections. Presentation = data format/translation/encoding. Drill this distinction.
