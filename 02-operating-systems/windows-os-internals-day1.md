# Windows OS Internals — SOC Analyst Notes (Day 1)

## Why OS Internals Matter for SOC Work

Every attack that lands on an endpoint leaves traces in the operating system — process creation logs, failed logins, privilege changes, scheduled tasks. A SOC analyst who can't read Windows logs is blind to 80% of what happens after an attacker gets past the perimeter. This tier is about understanding what normal looks like so you can spot what isn't.

---

## Event Viewer — Your Primary Windows Investigation Tool

**How to open:** `Windows key + R` → type `eventvwr.msc` → Enter (run as Administrator)

### Windows Logs Structure

| Log | What it records |
|---|---|
| **Application** | Events from applications and programs running on the system |
| **Security** | Login attempts, privilege use, account changes — most important for SOC |
| **Setup** | Windows installation and update events |
| **System** | OS events — driver failures, service starts/stops, hardware issues |
| **Forwarded Events** | Logs collected from other machines (enterprise environments) |

**Security log is the primary SOC focus** — everything authentication, process, and privilege-related lives here.

---

## Critical Windows Event IDs — Memorize These

| Event ID | What it means | SOC relevance |
|---|---|---|
| **4624** | Successful logon | Baseline — watch for unexpected accounts, times, logon types |
| **4625** | Failed logon | Volume = brute force, pattern across accounts = password spray |
| **4634** | Logoff | Tracks session duration |
| **4648** | Logon with explicit credentials | `runas` — possible lateral movement |
| **4672** | Special privileges assigned to new logon | Admin logon — watch for unexpected accounts |
| **4688** | New process created | Every program that runs — critical for detecting malware execution |
| **4698** | Scheduled task created | Attackers create tasks for persistence |
| **4720** | User account created | New account = possible backdoor |
| **4732** | User added to privileged group | Privilege escalation indicator |
| **4776** | Credential validation | Authentication against local SAM database |
| **4771** | Kerberos pre-auth failed | Failed domain logon — like 4625 for domain environments |

---

## Logon Types — Know These Cold

Found inside 4624 and 4625 events. The logon type tells you *how* someone authenticated — which is often more important than *who.*

| Type | Name | What it means | SOC relevance |
|---|---|---|---|
| **2** | Interactive | Physical keyboard login at the machine | Normal for users, suspicious at odd hours |
| **3** | Network | Authenticated over network (file shares, mapped drives) | Common, but watch for unusual source IPs |
| **4** | Batch | Scheduled task ran | Check if task is legitimate |
| **5** | Service | Windows service started | Normal background activity |
| **7** | Unlock | Screen unlocked after lock screen | Normal user behavior |
| **8** | NetworkCleartext | Network logon with plaintext credentials | **Always suspicious** — unencrypted credentials |
| **9** | NewCredentials | `runas` with different credentials | Could indicate lateral movement |
| **10** | RemoteInteractive | RDP login | Watch carefully — common attack vector |
| **11** | CachedInteractive | Login with cached credentials (no DC available) | Note if happens unexpectedly |

**Most important to watch: Type 3 and Type 10.**
- Type 3 from unexpected external IP = possible lateral movement
- Type 10 = RDP login = common ransomware entry point

---

## Reading a Real 4624 Event — Lab Observation

**Event captured on actual Windows machine (Tapiwa's laptop):**

```
Event ID: 4624 — Successful Logon
Account Name: TAPIWA$
Account Domain: WORKGROUP
Logon Type: 5 (Service)
Source Network Address: -
```

**Decoded:** Machine account (`$` suffix = computer account, not human), service logon (Type 5 = background Windows service starting), no network source (happened locally). Completely benign — Windows background process authenticating itself.

**Key detail — dollar sign suffix:** `TAPIWA$` with `$` = machine/computer account. `TAPIWA` without `$` = human user account. This distinction matters when reading logs — machine accounts doing service logons is normal noise; human accounts doing unexpected service logons is suspicious.

---

## Reading a Real 4625 Event — Lab Observation

**Event captured on actual Windows machine:**

```
Event ID: 4625 — Failed Logon
Account Name: TAPIWA$
Logon Type: 2 (Interactive)
Source Network Address: 127.0.0.1
```

**Decoded:** Machine account failed an interactive logon attempt, originating from loopback (127.0.0.1 = the machine itself). Benign — Windows internal process with slightly mismatched credentials (common after updates/password changes).

**4625 becomes dangerous through volume and pattern:**

| Pattern | What it suggests |
|---|---|
| Single 4625 then 4624 | User mistyped password — normal |
| 50+ 4625s in 60 seconds, same account | Brute force against that account |
| 4625s spread across many different accounts | Password spraying |
| 4625s from external IP, Type 3 or Type 10 | Remote brute force (SSH/RDP attack) |

---

## 4688 — New Process Created (Most Powerful Detection Event)

Every single program that runs on Windows generates a 4688. Malware has to execute somehow — 4688 catches it.

**Enabling 4688 on Windows Home (secpol.msc not available):**
```cmd
# Run Command Prompt as Administrator, then:
auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable

# Verify:
auditpol /get /subcategory:"Process Creation"
# Should show: Success and Failure
```

### Key Fields in a 4688 Event

| Field | What it tells you |
|---|---|
| **New Process Name** | Full path of the program that launched |
| **Creator Process Name** | What spawned it — the parent process |
| **Token Elevation Type** | Whether it ran with admin privileges |
| **Account Name** | Which user account it ran under |

### Token Elevation Types

| Type | Value | Meaning |
|---|---|---|
| TokenElevationTypeDefault | 1 | No elevation — normal user permissions |
| **TokenElevationTypeFull** | **2** | **Full admin token — UAC elevated** |
| TokenElevationTypeLimited | 3 | Admin account running without elevation |

---

## Real 4688 Events Captured — Lab Observations

### Event 1: Event Viewer launching itself

```
New Process Name:     C:\Windows\System32\mmc.exe
Creator Process Name: C:\Windows\explorer.exe
Token Elevation Type: TokenElevationTypeFull (2)
Mandatory Label:      High Mandatory Level
Account:              TAPIWA\Maramba
```

**Decoded:** User launched Event Viewer (mmc.exe) from the desktop (explorer.exe) with admin elevation. Normal — Event Viewer requires admin rights to read Security logs.

### Event 2: UAC prompt appearing

```
New Process Name:     C:\Windows\System32\consent.exe
Creator Process Name: C:\Windows\System32\svchost.exe
Token Elevation Type: TokenElevationTypeDefault (1)
Target Subject:       NULL SID
```

**Decoded:** A Windows service (inside svchost.exe) triggered a UAC elevation prompt (consent.exe). Normal — happens when services need elevated permissions. The NULL SID target means the UAC decision hasn't been made yet.

---

## The Parent-Child Process Relationship — Core Detection Skill

**The chain to always read:**
```
Creator Process → New Process → Elevation Type → Account
```

### Normal parent-child relationships

| Parent | Child | Verdict |
|---|---|---|
| `explorer.exe` | `notepad.exe` | Normal — user opened Notepad |
| `services.exe` | `svchost.exe` | Normal — Windows service host starting |
| `explorer.exe` | `mmc.exe` | Normal — user opened management console |

### Suspicious parent-child relationships

| Parent | Child | Why suspicious |
|---|---|---|
| `winword.exe` | `powershell.exe` | Word has no legitimate reason to spawn PowerShell |
| `outlook.exe` | `cmd.exe` | Email client spawning command prompt = phishing execution |
| `explorer.exe` | `regsvr32.exe` downloading from internet | Living-off-the-land technique |
| `cmd.exe` | `mimikatz.exe` | Credential dumping |

---

## svchost.exe — Know This Process Cold

One of the most common and most abused processes on Windows.

**Legitimate svchost.exe always:**
- Lives at `C:\Windows\System32\svchost.exe` — anywhere else = malware
- Is spawned by `services.exe` as parent — any other parent = suspicious
- Runs as SYSTEM, LOCAL SERVICE, or NETWORK SERVICE — not as regular user

**Attacker technique:** name malware `svchost.exe` or inject code into legitimate svchost processes to blend into normal process lists. Path and parent process are what expose the fake.

---

## Attack Scenario — Macro-Enabled Document (Phishing)

**4688 chain that indicates a phishing attack:**

```
Creator: C:\Windows\System32\winword.exe
New Process: C:\Windows\System32\powershell.exe
Elevation: TokenElevationTypeFull (2)
Account: regular_user@company.com
```

**What happened:** victim opened a malicious Word document delivered via phishing email. The document contained a **VBA macro** (Visual Basic for Applications — Office's built-in scripting language) that executed automatically on open. The macro spawned an elevated PowerShell process to download or execute a second-stage payload.

**Why suspicious:** Word and PowerShell have no legitimate parent-child relationship. Word is a document editor; PowerShell is a system command execution engine. This relationship in logs = near-certain malware execution.

**Why elevation makes it worse:** Type 2 means the attacker's code ran with full admin privileges — can install software, modify system files, create persistence mechanisms, disable security tools.

**Full attack chain:**
```
Phishing email → malicious .docx attachment
    ↓
Victim opens document
    ↓
VBA macro auto-executes
    ↓
Macro calls PowerShell
    ↓
PowerShell downloads/executes malware
    ↓
4688 log: winword.exe → powershell.exe (elevated)

```

**MITRE ATT&CK mapping (to revisit in Frameworks tier):**
- T1566.001 — Phishing: Spearphishing Attachment (delivery)
- T1059.001 — Command and Scripting Interpreter: PowerShell (execution)

---
**Quick Things to note which were left out :**
-Event ID 4104 (PowerShell Script Block Logging). My own phishing scenario at the bottom ends at "PowerShell downloads/executes malware" — but 4688 only tells you PowerShell launched, not what it did. 4104 is what shows the actual commands run inside that PowerShell session. Without it, you can see the door opened but not what happened in the room. Worth a short addition, not a new file.
-Event ID 1102 (audit log cleared). This is the "attacker covering their tracks" event — one of the most commonly expected Event IDs because it's the signal that something was deliberately hidden

## True/False Positive Reminder (from networking tier, reinforced here)

| Term | Meaning |
|---|---|
| **True Positive** | Alert fired, threat is real |
| **False Positive** | Alert fired, no real threat |
| **True Negative** | No alert, no threat — correct outcome |
| **False Negative** | No alert, threat IS real — worst case |

**Error caught this session:** called a confirmed brute force scenario a "true negative" — correct term is **true positive** (alert fired, threat confirmed real). Don't mix these up — wrong terminology in a SOC context means miscommunicating severity to the team.
