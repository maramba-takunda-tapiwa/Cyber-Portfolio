# SOC Detection Lab — Attack 1: Port Scan & Open Bindshell Exploitation

## Overview

First attack in the "Catch the Attacker" project — playing both attacker and defender against a deliberately vulnerable Metasploitable2 target from Kali Linux. This report documents the attack chain, the evidence captured, and the detection logic a real SOC analyst would apply.

---

## Attack Chain Summary

| Step | Action | MITRE ATT&CK Mapping |
|---|---|---|
| 1 | Service version scan (Nmap) | **T1046** — Network Service Discovery |
| 2 | Connected to exposed unauthenticated root shell | **T1133 / T1078** — External Remote Services / Valid Accounts (no-auth variant) |
| 3 | Confirmed root access | **Privilege already maximal — no escalation needed** |

---

## Step 1: Reconnaissance — Nmap Service Scan

**Command:**
```bash
nmap -sV 192.168.56.101
```

**Result (key findings):**

| Port | Service | Version | Risk Level |
|---|---|---|---|
| 21 | FTP | vsftpd 2.3.4 | **Critical** — known public backdoor vulnerability |
| 23 | Telnet | Linux telnetd | High — plaintext credential transmission |
| 512/513/514 | rexec/rlogin/rsh | Netkit | High — legacy, effectively unauthenticated |
| 1099 | java-rmi | GNU Classpath grmiregistry | High — known RCE vulnerability |
| **1524** | **bindshell** | **"Metasploitable root shell"** | **Critical — unauthenticated open root shell** |
| 3306 | MySQL | 5.0.51a-3ubuntu5 | Unknown yet — requires further testing, not confirmed vulnerable from scan alone |
| 6667 | IRC | UnrealIRCd | High — known backdoor in this version |

Full scan completed in 12.66 seconds against 1,000 ports, identifying 23 open services total.

**Severity reasoning:** initial instinct was to flag Telnet as most critical due to plaintext credentials. Corrected reasoning: **severity should be judged by how much effort stands between the attacker and impact, not just "this could theoretically be misused."** Telnet requires an attacker to actively intercept traffic to gain value (passive risk). The bindshell on port 1524 requires zero effort — Nmap's own service banner ("Metasploitable root shell") indicates direct, unauthenticated root access. Effort-to-impact ratio makes the bindshell the higher actual priority despite Telnet's well-known risks.

---

## Step 2: Exploitation — Connecting to the Bindshell

**Command:**
```bash
nc 192.168.56.101 1524
```

**Result:**
```
root@metasploitable:/# whoami
root
```

No username prompt. No password prompt. Immediate root shell access on first connection — confirming exactly what the Nmap service banner indicated. No exploit technique, payload, or credential brute-forcing was required; the service itself provided unauthenticated root access by design (deliberately planted vulnerability in Metasploitable2 for training purposes).

---

## Detection Analysis — Where This Should Have Been Caught

### The exploitation step (Step 2) is NOT the detectable moment

A single TCP connection to one port, with no errors, no failed attempts, no unusual payload signature — looks unremarkable in isolation against normal network traffic volume. By the time root access is achieved, the incident is no longer preventable, only discoverable through after-the-fact log/forensic review.

### The actual detectable moment: Step 1, the port scan itself

A port scan generates a distinct, identifiable pattern:
- **One source IP**
- **Many destination ports**
- **Short time window** (12.66 seconds for 1,000 ports in this case)
- **High volume of SYN packets**, many receiving no response (closed ports) or RST responses

This is not normal client behavior — no legitimate user or application connects to dozens of sequential/random ports on a single host in seconds. This is the **reconnaissance phase of the attack lifecycle**, and it is the one point in this entire chain where detection could have prevented everything that followed.

### What should trigger a SOC alert

| Indicator | Why it matters |
|---|---|
| High volume of SYN packets from a single source IP to multiple destination ports in a short window | Classic port scan signature — should trigger an IDS/IPS alert (e.g. Snort/Suricata) |
| Connection attempts to historically unused/legacy ports (512, 513, 514, 1524, etc.) | Legitimate traffic rarely touches these; any connection attempt is itself suspicious on a hardened network |
| Successful connection to a port with no authentication handshake at the application layer | Should be flagged by EDR/host monitoring even if the network-level connection itself looked unremarkable |

---

## Corrected Reasoning From This Session

**Initial assumption:** MySQL (port 3306) was picked as a high-priority target because SQL knowledge suggested it as an attack avenue.
**Correction:** Nmap output alone (open port + version) is reconnaissance data, not confirmed exploitability. Without further testing (credential attempts, known CVE lookup for that exact version), MySQL cannot be ranked above findings with direct evidence of exposure (like the bindshell's explicit "root shell" banner). Severity ranking requires evidence-based reasoning, not assumption based on familiarity with a technology.

**Refined terminology:** Nmap reveals what is "open and running" — this is **reconnaissance/service discovery**, not vulnerability discovery. Vulnerability identification is a separate, subsequent step (matching service versions against known CVEs). Conflating the two stages is a common beginner imprecision worth correcting early.

---

## Lab Evidence Captured
- Full Nmap scan output (23 open ports identified, services and versions documented above)
- Wireshark packet capture of the scan traffic (SYN packet pattern across multiple ports)
- Terminal session showing unauthenticated bindshell connection and `whoami` confirmation

---

## Next Steps
- **Attack 2:** SSH brute force (Hydra) — credential attack pattern and detection
- **Attack 3:** FTP backdoor exploitation (vsftpd 2.3.4) — known CVE exploitation
- Eventual goal: write basic Sigma detection rule for the port scan pattern identified here
