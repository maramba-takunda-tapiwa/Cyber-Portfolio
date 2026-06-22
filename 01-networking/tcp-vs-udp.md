# TCP vs UDP — SOC Analyst Notes

Both protocols live at **Layer 4 (Transport)**. Both move data between machines — but completely differently, and the difference matters for SOC work.

---

## TCP — Transmission Control Protocol

| | |
|---|---|
| **What it does** | Reliable, connection-based. A connection must be established before data is sent. Data is guaranteed to arrive, in order, without corruption |
| **Attacker use** | SYN flood (exploiting the handshake), port scanning, most malware C2 communication (reliability matters to attackers too) |
| **Defender detects** | Incomplete handshakes in firewall logs, unusual persistent connections to foreign IPs, high-volume SYN with no ACK |
| **In logs** | `SYN → SYN-ACK → ACK` completing = normal. `SYN → SYN → SYN` with no ACK = SYN flood |

### The 3-Way Handshake

```
Client          Server
  |---- SYN ------>|   "I want to connect"
  |<-- SYN-ACK ----|   "Acknowledged, ready?"
  |---- ACK ------>|   "Ready. Connection established."
  |   [DATA FLOWS]  |
```

**Why attackers exploit it:** during a SYN flood, the attacker sends thousands of SYNs but never sends the final ACK. The server allocates memory for each half-open connection waiting for an ACK that never comes, eventually exhausting resources.

---

## UDP — User Datagram Protocol

| | |
|---|---|
| **What it does** | Fast, connectionless. No handshake, no delivery guarantee, no ordering |
| **Attacker use** | UDP flood DDoS, DNS attacks (DNS runs on UDP), DHCP starvation, exfiltration (harder to track than TCP) |
| **Defender detects** | Massive UDP volume from a single source, suspicious DNS queries, unusual ports receiving UDP traffic |
| **In logs** | `UDP flood: 50,000 packets/sec from 185.220.x.x to port 53` — DNS DDoS or DNS tunnelling |

---

## TCP vs UDP — Direct Comparison

| Feature | TCP | UDP |
|---|---|---|
| Connection | Established first (handshake) | None needed |
| Reliability | Guaranteed delivery | No guarantee |
| Speed | Slower | Faster |
| Order | In order | No ordering |
| Use cases | HTTP, SSH, FTP, email | DNS, VoIP, streaming, gaming |
| Attack surface | SYN flood, session hijack | UDP flood, DNS attacks |

---

## DNS Tunnelling — Why it matters

Most malware C2 uses TCP for reliability. But attackers increasingly use **DNS over UDP** for exfiltration because DNS traffic is rarely blocked and blends in with normal traffic.

- Normal DNS query: short, resolves a known domain, occasional
- DNS tunnelling: high frequency, unusually long subdomains (e.g. `aGVsbG8gd29ybGQ=.exfil.attacker.com`), large response sizes
- The long subdomain is the tell — attackers encode stolen data **into the subdomain itself**, smuggled out chunk by chunk, query by query

---

## Self-Test — Scenarios

**Scenario 1:** Firewall log shows 10,000 SYN/sec from a single IP to a web server, zero ACK responses.
- **My answer:** TCP, SYN flood, Layer 4 (Transport), single source = DoS (not DDoS, since DDoS implies multiple sources)
- **Correction received:** technically correct on protocol/attack/layer. On response — as a Tier 1 analyst, the proper first move is usually to **verify it's not a false positive, then escalate per playbook** rather than unilaterally blocking the IP. Tier 1 often doesn't have direct firewall authority; escalation is the safer default action to learn.

**Scenario 2:** SIEM alert — high volume UDP on port 53 to a domain with a 200-character subdomain.
- **My answer:** UDP, DNS tunnelling, port 53 chosen because DNS traffic is rarely blocked and treated as normal — correct.
- **Added detail:** the long subdomain isn't random noise — it's literally the encoded stolen data being exfiltrated piece by piece.

**Key lesson:** Know the technical answer, but also think about *role-appropriate action*. Tier 1 = triage and escalate. Tier 2/3 = investigate and act. Don't jump straight to "block it" without knowing what your actual authority is.
