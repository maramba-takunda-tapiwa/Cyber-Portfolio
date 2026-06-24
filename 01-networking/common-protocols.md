# Common Protocols — SOC Analyst Notes

Every protocol has a default port, a purpose, and a typical risk profile. SOC analysts need to instantly recognize what's normal vs abnormal for each one — attackers often hide in plain sight using legitimate protocols, or abuse insecure ones directly.

---

## HTTP / HTTPS (Port 80 / 443)

| | |
|---|---|
| **What it does** | Web traffic. HTTP = plaintext, HTTPS = encrypted via TLS |
| **Attacker use** | Malicious downloads, C2 communication disguised as web traffic, credential theft over HTTP |
| **Defender detects** | Web proxy/firewall logs, unusual user-agents, traffic to known-bad domains, large outbound data over HTTPS (possible exfil) |
| **In logs** | Repeated POST requests to an unfamiliar external IP at regular intervals = possible C2 beaconing |

---

## FTP (Port 21)

| | |
|---|---|
| **What it does** | File Transfer Protocol — uploading/downloading files between systems |
| **Attacker use** | Anonymous FTP login abuse, plaintext credential sniffing, malware staging/exfil |
| **Defender detects** | Anonymous logins in FTP logs, large file transfers to external IPs, FTP traffic from unusual internal hosts |
| **In logs** | `USER anonymous` followed by successful login = serious misconfiguration |

---

## SSH (Port 22)

| | |
|---|---|
| **What it does** | Secure encrypted remote access/terminal to a system |
| **Attacker use** | Brute forcing weak SSH credentials, lateral movement once inside a network, tunnelling other traffic through SSH to evade detection |
| **Defender detects** | Repeated failed login attempts from same/different IPs, logins at unusual hours, logins from unfamiliar geographic locations |
| **In logs** | `Failed password for root from 185.220.x.x` repeated hundreds of times = brute force attempt |

---

## SMTP (Port 25)

| | |
|---|---|
| **What it does** | Sends email between mail servers |
| **Attacker use** | Phishing, spam relay abuse, BEC (Business Email Compromise), spoofed sender addresses |
| **Defender detects** | SPF/DKIM/DMARC failures, unusual outbound email volume, mail server suddenly relaying mail for external domains |
| **In logs** | Sudden spike in outbound emails from a single account = compromised mailbox being used to spam |

---

## DHCP (Port 67/68)

| | |
|---|---|
| **What it does** | Automatically assigns IP addresses to devices joining a network |
| **Attacker use** | DHCP starvation (exhausting the IP pool, denying service to legit devices), rogue DHCP server handing out malicious gateway/DNS info |
| **Defender detects** | DHCP snooping on switches, monitoring for unauthorized DHCP servers responding to requests |
| **In logs** | Two DHCP servers responding to the same client request = rogue DHCP server present |

---

## ARP (No port — Layer 2)

Resolves IP addresses to MAC addresses on a local network. Its complete lack of authentication is exactly why ARP spoofing works so easily. (Covered in depth in OSI model notes — Layer 2.)

---

## SMB (Port 445)

| | |
|---|---|
| **What it does** | Windows file/printer sharing protocol |
| **Attacker use** | Lateral movement (EternalBlue/WannaCry exploited this), ransomware spreading across a network, credential relay attacks |
| **Defender detects** | Unusual SMB traffic between workstations (should mostly be client→server, not workstation→workstation), failed auth attempts |
| **In logs** | Sudden SMB connections between dozens of workstations simultaneously = potential ransomware spreading |

---

## Quick Reference Table

| Protocol | Port | Encrypted? | Primary Risk |
|---|---|---|---|
| HTTP | 80 | No | Plaintext data, malicious downloads, C2 |
| HTTPS | 443 | Yes (TLS) | C2 hidden in encrypted traffic, exfil |
| FTP | 21 | No | Anonymous access, credential sniffing |
| SSH | 22 | Yes | Brute force, tunnelling for evasion |
| SMTP | 25 | No (by default) | Phishing, spam relay, BEC |
| DHCP | 67/68 | No | Starvation attacks, rogue servers |
| ARP | N/A (L2) | No | Spoofing, MITM |
| SMB | 445 | Varies | Lateral movement, ransomware spread |

---

## Not Yet Covered (parked for later sessions)

- **Telnet (23)** — insecure remote access, red flag if seen at all in modern traffic
- **RDP (3389)** — Windows Remote Desktop, major brute force / ransomware lateral movement target
- **NTP (123)** — time sync, relevant for log accuracy and as a DDoS amplification vector
- **LDAP (389)** — Active Directory authentication, relevant once going deeper into AD attacks
- **SNMP (161)** — network device monitoring, relevant for network-focused attacks

---

## Up Next
**DNS Deep Dive** — record types, DNS resolution flow, and DNS-specific attacks (tunnelling, spoofing, hijacking, fast flux, DGA, typosquatting). Starting tomorrow.
