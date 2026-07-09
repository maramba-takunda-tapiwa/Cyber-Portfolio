# Windows Registry & Services — SOC Analyst Notes

## Windows Registry

### What it Actually Is

The Registry is Windows' central database — it stores configuration settings for literally everything on the system: the OS itself, every installed application, user preferences, hardware drivers, startup programs, and security policies. Think of it as Windows' memory of how everything is supposed to behave.

**How to open:** `Windows key + R` → type `regedit` → Enter (as Administrator)

---

### Registry Structure — The Five Hives

| Hive | Abbreviation | What it stores |
|---|---|---|
| **HKEY_LOCAL_MACHINE** | HKLM | System-wide settings — hardware, OS, installed software, services. Applies to all users |
| **HKEY_CURRENT_USER** | HKCU | Settings for the currently logged-in user only |
| **HKEY_USERS** | HKU | Settings for all user profiles on the machine |
| **HKEY_CLASSES_ROOT** | HKCR | File associations and COM object registrations |
| **HKEY_CURRENT_CONFIG** | HKCC | Current hardware profile information |

**Most important for SOC work: HKLM and HKCU**

---

### Registry Keys SOC Analysts Must Know

#### Autorun Locations — Where Malware Hides for Persistence

Anything listed here launches automatically every time Windows starts. First place to check during a malware investigation.

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
```

**HKLM vs HKCU for persistence:**
- HKLM autoruns require **admin rights** to write — harder for malware without privilege escalation
- HKCU autoruns can be written by **any regular user process** — easier for malware to abuse, no UAC triggered

#### Other High-Value Locations

| Key | What's there | Attack relevance |
|---|---|---|
| `HKLM\SYSTEM\CurrentControlSet\Services` | Every registered Windows service | Malware installing itself as a service appears here |
| `HKLM\SAM` | Local user account password hashes | Mimikatz targets this for credential dumping |
| `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\RecentDocs` | Recently opened files | Forensic evidence of what a user accessed |
| `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` | Login settings | Attackers modify this to auto-run malware at login |

---

### Real Lab Observation — Autorun Baseline (Tapiwa's Laptop)

#### HKLM\...\Run — System-wide autoruns

| Name | Path | Verdict |
|---|---|---|
| RtkAudUService | `C:\Windows\System32\DriverStore\FileRepository\...` | Realtek Audio driver — legitimate |
| SecurityHealth | `%windir%\system32\SecurityHealthSystray.exe` | Windows Defender system tray — legitimate |
| WavesSvc | `C:\Windows\System32\DriverStore\FileRepository\...` | Waves audio software — legitimate |

**Result: Clean** — all three entries point to `System32` or DriverStore, all recognizable audio/security software.

#### HKCU\...\Run — User-level autoruns

| Name | Path | Verdict |
|---|---|---|
| Docker Desktop | `C:\Program Files\Docker\...` | Legitimate |
| Mathworks Servi... | `C:\Users\maram\AppData\Local\MathWorks\...` | MATLAB — legitimate |
| OneDrive | `C:\Program Files\Microsoft OneDrive\...` | Legitimate |
| Opera GX Brows... | `C:\Users\maram\AppData\Local\Programs\Opera...` | Legitimate |
| Opera GX Stable | `C:\Users\maram\AppData\Local\Programs\Opera...` | Legitimate |
| Opera Stable | `C:\Users\maram\AppData\Local\Programs\Opera...` | Legitimate |
| Teams | `C:\Users\maram\AppData\Local\Microsoft\...` | Legitimate |
| ut | `C:\Users\maram\AppData\Roaming\utorrent\uTor...` | uTorrent — legitimate but worth noting |

**Result: Clean** — all recognizable software.

**Notable observations:**
- Three separate Opera entries all launching at startup — unnecessary startup load, not malicious
- uTorrent (`ut`) — legitimate but in corporate environments would be flagged; torrent clients used for data exfiltration and often bundled with PUPs

---

### Red Flags in Autorun Entries

| Indicator | Why suspicious |
|---|---|
| Path pointing to `%TEMP%` or `AppData\Roaming` with random name | Malware drops itself in temp/roaming folders |
| Entry with random character name (e.g. `xk3f9a`) | Malware uses random names to avoid detection |
| Path pointing to `.vbs`, `.ps1`, or `.bat` file | Script-based persistence |
| Path in System32 but filename isn't a known Windows binary | DLL hijacking or masquerading |

---

### Key Principle — Baseline First

Before you can spot malicious autorun entries, you need to know what legitimate looks like on that specific machine. Baseline-building is exactly what you'd do at the start of a real investigation — establish normal, then look for anomalies.

---

## Windows Services

### What Services Are

Services are background processes that run independently of user sessions — they start at boot, run as specific accounts, and perform system-level tasks. They're a major persistence mechanism because they survive reboots automatically.

**How to open:** `Windows key + R` → type `services.msc` → Enter

---

### Startup Types

| Startup Type | Meaning | SOC relevance |
|---|---|---|
| **Automatic** | Starts every Windows boot | Should be legitimate, well-known services only |
| **Automatic (Delayed)** | Starts shortly after boot | Same — delayed for performance |
| **Automatic (Trigger)** | Starts when specific event triggers it | Normal — event-driven |
| **Manual** | Only starts when explicitly called | Normal on-demand services |
| **Disabled** | Cannot start at all | Sometimes used to lock down unused services |

**SOC focus:** any service set to **Automatic** that you don't recognize = persistence mechanism worth investigating.

---

### Four-Point Service Verification Checklist

When investigating any suspicious service, check all four:

```
1. PATH     → does it point to System32 or Program Files?
2. NAME     → is it recognizable or random characters?
3. DESC     → does it have a description, and does it make sense?
4. LOG ON   → what account is it running as?
```

---

### Service Account Types

| Account | Privilege level | Used for |
|---|---|---|
| **Local System** | Highest — full local admin, network as computer account | Core Windows services needing maximum access |
| **Local Service** | Low — minimal permissions, no network access | Services not needing much access |
| **Network Service** | Low local — can access network | Services needing network but not local admin |
| **Custom user account** | Depends on account | Applications needing specific permissions |

**Least privilege principle:** a service should run as the lowest-privilege account that allows it to function. A service running as Local System when it only needs Local Service is a misconfiguration — and an attacker running their backdoor as Local System is exploiting exactly this.

---

### Real Lab Observation — Base Filtering Engine (BFE)

```
Service name:         BFE
Display name:         Base Filtering Engine
Description:          Manages firewall and Internet Protocol security policies
Path to executable:   C:\Windows\system32\svchost.exe -k LocalServiceNoNetworkFirewall -p
Startup type:         Automatic
Service status:       Running
Log on as:            Local Service
```

**Verification result:**
```
✅ Path    → C:\Windows\system32\svchost.exe (legitimate System32 location)
✅ Name    → BFE (recognizable Windows service, well-documented)
✅ Desc    → Clear, accurate description of what it does
✅ Log On  → Local Service (least privilege — correct for a firewall engine)
```

**Why svchost.exe here is normal:** Windows hosts many services inside svchost.exe processes. The `-k LocalServiceNoNetworkFirewall` flag tells svchost which service group to load. This is standard Windows architecture.

**If BFE stops running:** Windows Firewall silently disables. Attackers sometimes stop this service specifically to remove firewall protection without the user noticing.

---

### svchost.exe — The Most Abused Legitimate Process

**Legitimate svchost.exe always:**
- Lives at `C:\Windows\System32\svchost.exe` — anywhere else = malware
- Is spawned by `services.exe` as parent — any other parent = suspicious
- Runs as SYSTEM, LOCAL SERVICE, or NETWORK SERVICE — not a regular user account

**Attacker technique:** name malware `svchost.exe` or inject malicious code into legitimate svchost processes to blend into normal process lists. Path and parent process are what expose the fake.

---

### Red Flags in Services

| Red flag | Why suspicious |
|---|---|
| Service name is random characters | Malware uses random names to avoid detection |
| Path points to `%TEMP%` or user `AppData` | Legitimate services live in System32 or Program Files |
| Description is blank or generic ("Windows Service") | Malware often has no description or a vague one |
| New Automatic service that appeared recently | Check install date against incident timeline |
| Service running as a named user account | Attacker-controlled account maintaining persistent access |
| Service name one character off a real Windows service | **Masquerading** — deliberate deception |

---

### Masquerading — The Sophisticated Red Flag

**Example caught in session:**
```
Service name: wcmsvcs
Path: C:\Users\maram\AppData\Roaming\wcmsvcs.exe
Startup type: Automatic
Log on as: Local System
Description: Windows Connection Manager
```

**Red flags identified:**
```
❌ Path    → AppData\Roaming (not System32 — malware territory)
❌ Name    → wcmsvcs (one letter off from real service wcmsvc — deliberate masquerading)
❌ Log On  → Local System (highest privilege — unnecessary for claimed purpose)
❌ Startup → Automatic (persistence — survives every reboot)
```

**The masquerading detail is the most sophisticated:** attackers name malware after real Windows services with one character changed, hoping analysts glance at it and assume it's legitimate. Always Google the exact service name — if it's one character off from a real service, that's deliberate deception.

**Lesson:** don't just check path and privileges — verify the exact name against known Windows services.
