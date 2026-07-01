# Firewalls, Routing, VPNs & Proxies — SOC Analyst Notes

## Firewalls

### What it does
A firewall sits between two networks (most commonly between your internal network and the internet) and makes **allow or deny decisions** on traffic based on rules. Rules can be based on: source/destination IP, source/destination port, protocol (TCP/UDP/ICMP), direction (inbound vs outbound), and connection state.

---

### Types of Firewalls

#### 1. Packet Filtering Firewall (Stateless)

| | |
|---|---|
| **What it does** | Inspects individual packets in isolation — checks source IP, destination IP, port, protocol against a ruleset. Each packet judged independently, no memory of previous packets |
| **Attacker use** | Easy to evade — craft packets that match allowed rules even if part of a malicious flow, because the firewall has no context about the overall connection |
| **Defender detects** | Blocks obvious bad IPs/ports but misses sophisticated attacks hiding inside allowed traffic |
| **In logs** | `DENY TCP 185.220.x.x:49231 → 10.0.0.5:22` — blocked by IP/port rule |

#### 2. Stateful Inspection Firewall

| | |
|---|---|
| **What it does** | Tracks the **state** of active connections — knows whether a packet is part of an established legitimate TCP handshake or an out-of-context packet trying to sneak through |
| **Attacker use** | Harder to evade — can't spoof a packet to look legitimate if the firewall knows no handshake ever happened for that session |
| **Defender detects** | Drops packets that don't belong to a known established connection — catches SYN-ACK packets arriving with no preceding SYN |
| **In logs** | `DENY TCP 10.0.0.5:80 → INVALID STATE` — packet arrived but no matching connection state exists |

#### 3. Next-Generation Firewall (NGFW)

| | |
|---|---|
| **What it does** | Everything stateful does, plus: deep packet inspection (looks inside payload not just headers), application awareness (blocks specific apps regardless of port), IDS/IPS integration, SSL inspection |
| **Attacker use** | Hardest to evade — even encrypted traffic can be inspected via SSL termination. Attackers may mimic legitimate application traffic patterns |
| **Defender detects** | Can block by application, not just port — malware on port 443 gets identified as non-HTTPS and blocked |
| **In logs** | `DENY Application: BitTorrent, User: john.doe, IP: 192.168.1.45` |

---

### Blacklist vs Whitelist Policy — Critical Distinction

| Approach | How it works | Real-world use |
|---|---|---|
| **Blacklist (default-accept)** | Allow everything, explicitly block known bad | Metasploitable's default — dangerous, requires knowing every threat in advance |
| **Whitelist (default-deny)** | Block everything, explicitly allow only what's needed | Production hardened systems — nothing gets through by accident |

**In iptables:** `-P INPUT DROP` sets default-deny. Only traffic matching an explicit ACCEPT rule gets through. Everything else falls through to the default policy and gets dropped.

---

### DROP vs REJECT

| | DROP | REJECT |
|---|---|---|
| **What happens** | Silently discards the packet — sender gets no response, waits for timeout | Sends back an error (TCP RST or ICMP unreachable) — sender immediately knows it was refused |
| **Attacker implication** | Introduces uncertainty — can't tell if port is filtered or just slow | Tells attacker definitively something is blocking them |
| **Real-world use** | Preferred for external-facing rules — less information given to attackers | Better for internal rules where legitimate users need fast feedback |

---

### Rule Order — First Match Wins

iptables processes rules **top to bottom** — the first rule that matches a packet applies, and processing stops. This means:

- **ACCEPT rules for trusted IPs must come BEFORE DROP rules**, otherwise the DROP rule matches first and the trusted IP gets blocked anyway
- **Default policy (`-P INPUT DROP`) is the fallback** — it only applies to packets that reach the end of the chain without matching any rule. It's not an active inspector; it's a last-resort bouncer.

```
Packet arrives →
  Rule 1: match? → ACCEPT/DROP, stop
  Rule 2: match? → ACCEPT/DROP, stop
  Rule 3: match? → ACCEPT/DROP, stop
  No rules matched → Default policy applies → DROP
```

---

## Routing

### What it does
Routing determines how packets travel from source to destination across networks. A **router** maintains a **routing table** — a map of which interface to send packets out of based on destination IP.

```
Routing table example:
Destination     Gateway         Interface
192.168.1.0/24  directly conn.  eth0
0.0.0.0/0       10.0.0.1        eth1   ← default route (everything else)
```

**Longest prefix match:** router picks the most specific matching rule. `/24` beats `/16` beats `/0`.

### Key Routing Concepts for SOC

| Concept | What it is | SOC relevance |
|---|---|---|
| **Default gateway** | Router all traffic goes to when no specific route matches | Changing a host's default gateway redirects all traffic through attacker — network-level MITM |
| **NAT** | Translates private IPs to public IP for internet access | Makes internal hosts invisible from internet — passive protection, NOT a security control by itself |
| **BGP hijacking** | Attacker announces fake routes to trick routers into sending traffic through them | Nation-state level — intercepts large volumes of internet traffic |
| **Route poisoning** | Injecting false routing table entries to redirect or drop traffic | Can cause network outages or MITM at routing layer |

---

## VPNs and Proxies

### VPN (Virtual Private Network)

| | |
|---|---|
| **What it does** | Creates an encrypted tunnel between client and VPN server — all traffic routed through that server, hiding real IP and encrypting traffic from anyone on the path |
| **Legitimate use** | Remote workers accessing internal network securely, bypassing geographic restrictions |
| **Attacker use** | Hiding real IP during attacks, bypassing IP-based blocklists, commercial VPNs to make attack traffic look legitimate |
| **Defender detects** | Traffic to known VPN provider IP ranges, geographic login anomalies, high-volume traffic to single external IP (the VPN endpoint) |
| **In logs** | User normally logs in from Hungary, suddenly logs in from Netherlands IP — flag for potential credential theft or VPN use |

### Proxy

| | |
|---|---|
| **What it does** | Intermediary between client and destination — client sends request to proxy, proxy forwards it, returns response |
| **Types** | Forward proxy (client → proxy → internet), Reverse proxy (internet → proxy → internal server, used for load balancing/hiding infrastructure) |
| **Attacker use** | Chaining multiple proxies to anonymize attack origin, using compromised machines as proxies, bypassing content filters |
| **Defender detects** | SOCKS/HTTP proxy connections to unexpected external IPs, connections to port 1080 (standard SOCKS port) |
| **In logs** | Internal host connecting to port 1080 on external IP at regular intervals = possible C2 traffic being proxied |

### VPN vs Proxy Comparison

| | VPN | Proxy |
|---|---|---|
| **Scope** | All traffic on the device (system-wide) | Usually application-level only |
| **Encryption** | Always encrypts | Usually doesn't encrypt — just relays |
| **Attacker preference** | Longer-term anonymization | Quick, lightweight, easy to chain |

---

## SOC Relevance — Why Firewall Logs Matter

Firewalls and routing produce the **primary evidence trail** for SOC analysts. When an attack happens:
- DENY entries show what the attacker tried but couldn't get through
- ALLOW entries show what got through — the critical ones to investigate
- Blocked attempts reveal attacker TTPs and reconnaissance patterns before they found an open path

A SOC analyst who can't read a firewall log is essentially blind.

---

## Firewall Practical — iptables Lab (Kali vs Metasploitable2)

### Starting state — Metasploitable default (completely open)

```bash
sudo iptables -L -v -n
```

```
Chain INPUT  (policy ACCEPT)   ← all incoming traffic accepted
Chain FORWARD (policy ACCEPT)  ← all forwarded traffic accepted
Chain OUTPUT (policy ACCEPT)   ← all outgoing traffic accepted
```

No rules. Zero restrictions. Every port, every protocol, every IP — accepted by default. This is exactly why port 1524 (bindshell) was sitting wide open in Attack 1.

---

### Task 1 — Block SSH from a specific source IP

```bash
# Verify SSH currently reachable from Kali
nc -zv 192.168.56.101 22   # Returns: open

# Add block rule on Metasploitable
sudo iptables -A INPUT -s 192.168.56.105 -p tcp --dport 22 -j DROP

# Test again from Kali
nc -zv 192.168.56.101 22   # Now hangs — silently dropped
```

**Command breakdown:**
- `-A INPUT` — append rule to INPUT chain
- `-s 192.168.56.105` — match traffic from Kali's IP only
- `-p tcp` — TCP protocol
- `--dport 22` — destination port 22 (SSH)
- `-j DROP` — silently discard, no response sent

**Result:** SSH from Kali blocked. DROP means attacker can't tell if port is filtered or just slow — no RST sent back.

---

### Task 2 — Block the bindshell port from any source

```bash
sudo iptables -A INPUT -p tcp --dport 1524 -j DROP

# Test from Kali
nc 192.168.56.101 1524   # Hangs — port unreachable
```

No `-s` flag = applies to all source IPs. The open root shell from Attack 1 is now unreachable from anywhere.

---

### Task 3 — Block ICMP (ping reconnaissance)

```bash
sudo iptables -A INPUT -s 192.168.56.105 -p icmp -j DROP

# Test from Kali
ping 192.168.56.101   # 100% packet loss — machine appears invisible
```

**Why this matters:** attackers ping a target first to confirm it's alive before attempting anything. Blocking ICMP removes that reconnaissance signal.

---

### Task 4 — Default-deny policy (real-world hardening)

**Critical lesson learned: rule order matters — first match wins.**

Wrong order (what was built during the session):
```
1. DROP tcp from Kali → port 22
2. DROP icmp from Kali
3. ACCEPT all from Kali   ← too late, DROP rules above already matched first
```
Result: even the trusted IP (Kali) gets blocked because DROP rules fire before the ACCEPT rule.

**Correct order — trusted IPs ACCEPT before any DROP rules:**

```bash
# Flush all existing rules
sudo iptables -F

# Set default-deny policy first
sudo iptables -P INPUT DROP

# Allow trusted IP (must come BEFORE any DROP rules)
sudo iptables -A INPUT -s 192.168.56.105 -j ACCEPT

# Explicit drops (actually redundant with default-deny, but kept for clarity)
sudo iptables -A INPUT -p tcp --dport 22 -j DROP
sudo iptables -A INPUT -p tcp --dport 1524 -j DROP
```

**Final ruleset:**
```
Chain INPUT (policy DROP)
ACCEPT  all  --  192.168.56.105  anywhere   ← trusted IP allowed first
DROP    tcp  --  anywhere → dpt:22
DROP    tcp  --  anywhere → dpt:1524
```

**Default policy behaviour:** `-P INPUT DROP` is NOT an active inspection rule — it's a fallback. Any packet that reaches the end of the chain without matching any rule gets dropped automatically. Nothing gets through unless explicitly ACCEPTed first.

---

### Key lessons from the practical

1. **Default-open is indefensible** — Metasploitable's out-of-the-box state (policy ACCEPT, no rules) is why every attack in this project worked with zero friction
2. **Rule order is everything** — first match wins, ACCEPT rules for trusted sources must precede DROP rules
3. **Default-deny makes specific DROP rules redundant** — if default policy is DROP, you only need ACCEPT rules for legitimate traffic; everything else fails automatically
4. **DROP vs REJECT is a deliberate choice** — DROP for external-facing rules (less attacker intelligence), REJECT for internal rules (faster feedback for legitimate users)
5. **Firewall rules are only as good as your understanding of normal traffic** — you can't write good ACCEPT rules if you don't know what your network's legitimate traffic looks like
