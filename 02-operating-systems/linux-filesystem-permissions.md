# Linux File System & Permissions — SOC Analyst Notes

## Why This Matters for SOC Work

Linux runs the majority of servers, cloud infrastructure, and network devices in enterprise environments. When an attacker compromises a Linux system, they leave traces in specific locations — knowing where things live and who can access them is what separates an analyst who finds the evidence from one who misses it entirely.

---

## Linux File System Structure

Unlike Windows (`C:\` as root), Linux has one root `/` and everything branches from it.

| Path | What lives there | SOC relevance |
|---|---|---|
| `/etc` | System configuration files | Attackers modify `/etc/passwd`, `/etc/cron*`, `/etc/hosts` for persistence |
| `/var/log` | System and application logs | **Primary investigation location** — all logs live here |
| `/tmp` | Temporary files, world-writable by anyone | Malware commonly drops itself here — no special permissions needed to write |
| `/home` | User home directories | User files, bash history, SSH keys |
| `/root` | Root user's home directory | High-value target — SSH keys, scripts, sensitive files |
| `/bin` and `/usr/bin` | Standard user commands | Unexpected files here = red flag |
| `/sbin` and `/usr/sbin` | System admin commands | Same — unexpected binaries are suspicious |
| `/proc` | Virtual filesystem — running processes | Live process information visible in real time |
| `/dev` | Device files | Attackers use `/dev/tcp` for network connections without normal tools |
| `/etc/cron*` | Scheduled tasks | Common persistence mechanism — attackers add malicious cron jobs here |

**Two most important locations for SOC day-to-day:**
- `/var/log` — where you investigate
- `/tmp` — where you look for dropped malware

---

## Linux Permissions

Every file and directory has three permission sets — **owner, group, others** — each with three possible permissions: **read (r), write (w), execute (x).**

```bash
ls -la /etc/passwd
-rw-r--r-- 1 root root 2847 Jun 25 10:23 /etc/passwd
```

### Reading the Permission String

```
- rw- r-- r--
↑  ↑   ↑   ↑
│  │   │   └── Others: read only
│  │   └────── Group: read only
│  └────────── Owner (root): read + write
└───────────── File type: - = file, d = directory, l = symlink
```

### Permission Values in Numbers

| Number | Permission | Meaning |
|---|---|---|
| 7 | rwx | Read + write + execute |
| 6 | rw- | Read + write |
| 5 | r-x | Read + execute |
| 4 | r-- | Read only |
| 0 | --- | No permissions |

`chmod 755 file` = owner gets rwx (7), group gets r-x (5), others get r-x (5).

---

## Critical Files — `/etc/passwd` vs `/etc/shadow`

These two files together form the Linux credential store. Their permissions are one of the first things to check on a compromised system.

### Lab Observation — Kali vs Metasploitable Comparison

**Kali:**
```
-rw-r--r-- 1 root root   3267 /etc/passwd   (644 — world readable, correct)
-rw-r----- 1 root shadow 1465 /etc/shadow   (640 — shadow group only, correct)
```

**Metasploitable:**
```
-rw-r--r-- 1 root root 1581 /etc/passwd   (644 — same, correct)
-rw-r----- 1 root root 1207 /etc/shadow   (640 — same permission string)
```

**Key difference:** Metasploitable's shadow file is owned by the `root` group instead of a dedicated `shadow` group — slightly weaker isolation. Any root-group process can read password hashes.

### What Each File Contains

| File | Contents | Readable by |
|---|---|---|
| `/etc/passwd` | Usernames, UIDs, home dirs, shell paths — NO passwords | Everyone (world-readable intentionally) |
| `/etc/shadow` | Actual password hashes | Root and shadow group only |

**Red flag:** if `/etc/shadow` shows `r--` for others (world-readable) — every user on the system can grab password hashes and crack them offline. Critical misconfiguration.

---

## Shadow File — Hash Format

```bash
sudo cat /etc/shadow
```

**Format of each line:**
```
username:$id$salt$hash:last_changed:min:max:warn:inactive:expire
```

**Hash type prefixes:**

| Prefix | Algorithm | Security |
|---|---|---|
| `$1$` | MD5 | **Weak — crackable in seconds/minutes** |
| `$2a$`/`$2b$` | bcrypt | Strong — designed to be slow |
| `$5$` | SHA-256 | Moderate |
| `$6$` | SHA-512 | Strong — modern Linux default |

**Lab finding — Metasploitable uses `$1$` (MD5) for all hashes:**
```
msfadmin:$1$XN10Zj2c$Rt/zzCW3mLtUWA.ihZja5/:14684:...
klog:$1$f22UMS4K$R9XkI.CmLdHhdUE3X9jqP0:14742:...
```

**Why this is trivially crackable — two reasons:**
1. MD5 is extremely fast to compute — modern GPUs try billions of combinations per second
2. Password `msfadmin` is a dictionary word and known default credential — appears in any decent wordlist immediately

**Dictionary attack:** cracking tool hashes wordlist entries using same algorithm (`$1$` MD5), compares results against stolen hash — finds match instantly when password is a common word.

**Accounts with `*` or `!`** (e.g. `www-data:*`) = login disabled, no hash to crack — service accounts not useful for direct login.

---

## Dangerous Permission Scenarios — Red Flags

| Scenario | Why dangerous |
|---|---|
| World-writable files in `/etc` | Anyone can modify system config |
| `/etc/shadow` world-readable | Every user can grab password hashes |
| **SUID bit set** (`s` instead of `x`) | File runs as its owner (often root) regardless of who executes it — privilege escalation vector |
| World-writable directories in system paths | Attacker drops malicious binaries that get executed by legitimate processes |

### Finding SUID Files

```bash
find / -perm -4000 -type f 2>/dev/null
```

Finds all files with SUID bit set. On Metasploitable this returns a suspicious number of files — each is a potential privilege escalation path.

---

## Key Takeaway

Before you can spot a misconfigured permission, you need to know what correct looks like. The baseline:

```
/etc/passwd  → -rw-r--r-- (644) — world readable, intentional
/etc/shadow  → -rw-r----- (640) — root + shadow group only
/tmp         → drwxrwxrwt (1777) — world writable with sticky bit
/etc         → drwxr-xr-x (755) — root writes, everyone reads
```

Anything deviating from this baseline on a production system = finding worth investigating.
