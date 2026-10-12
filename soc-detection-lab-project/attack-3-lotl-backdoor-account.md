# SOC Detection Lab — Attack 3: Living Off the Land — Backdoor Admin Account

## Overview

Third attack in the "Catch the Attacker" project, and the first on a different target — rather than Metasploitable2, this exercise was run on my own Windows laptop, in an elevated PowerShell session. Unlike Attacks 1 and 2, this isn't an exploit against a vulnerable service — it's a **Living Off the Land (LOTL)** technique: using only native, signed Windows tools (`net.exe`) to create a backdoor administrator account, with no malware and no file dropped. The goal was to prove the technique works with zero custom tooling, then find and interpret the evidence it leaves in the Windows Security log.

---

## Attack Chain Summary

| Step | Action | MITRE ATT&CK Mapping |
|---|---|---|
| 1 | Established baseline of existing local accounts | Reconnaissance — not independently mapped |
| 2 | Created a new local account (`hacker`) | **T1136.001** — Create Account: Local Account |
| 3 | Added the account to the local Administrators group | **T1098** — Account Manipulation |
| 4 | Confirmed persistence via `net user` | Verification — not independently mapped |
| 5 | Cleaned up: removed the account and group membership | N/A — remediation, not attack |

---

## Step 1: Baseline Recon

**Command:**
```powershell
net user
```

**Result:**
```
Administrator, DefaultAccount, Guest, Maramba, WDAGUtilityAccount, WsiAccount
```

Capturing the baseline first is what makes the later diff meaningful — without it, there's no way to prove `hacker` wasn't already there.

---

## Step 2: Create the Backdoor Account

**Command:**
```powershell
net user hacker Password234 /add
```

**Result:** Succeeded. A new local account `hacker` created using only a built-in Windows binary — no executable dropped to disk, no signature for antivirus to flag, since `net.exe` is legitimate, signed, and expected to be present on every Windows machine.

---

## Step 3: Escalate to Administrator

**Command:**
```powershell
net localgroup administrators hacker /add
```

**First attempt — failed:**
```
System error 1376 has occurred.
The specified local group does not exist.
```

**Second attempt (identical command) — succeeded.**

**Why this is worth documenting rather than skipping past:** the first failure was most likely a typo rather than anything meaningful, but the key lesson is that **failed commands leave their own trace** — an analyst reviewing this incident after the fact would see both the failed and successful attempts in sequence, which is itself useful context (e.g. ruling out that the attacker had the wrong group name memorized, or was testing group existence).

---

## Step 4: Confirm the Account

**Command:**
```powershell
net user
```

**Result:** `hacker` now appears in the account list, with administrator privileges — a fully functional backdoor admin account created using zero custom tooling.

---

## Step 5: Cleanup

Since this created a real local administrator account with a simple, guessable password on my own machine, it was removed immediately after the evidence was captured:

```powershell
net localgroup administrators hacker /delete
net user hacker /delete
net user    # confirmed 'hacker' is gone
```

The deletion itself also generates its own Security log event (**4726**), which is worth noting as part of the full lifecycle, not just the creation.

---

## Evidence — Windows Security Log

| Event | Event ID | What it confirms |
|---|---|---|
| Account creation (`net user hacker ... /add`) | **4720** | New local account created |
| Added to Administrators group | **4732** | Member added to a security-enabled local group |
| Account deletion (cleanup) | **4726** | Account removed |

**Key detail in the raw events:** both 4720 and 4732 list **Subject Account Name: Maramba, Account Domain: TAPIWA** — the events attribute the actions to the account that ran them, which in this case was genuinely me. Nothing in the event itself distinguishes "the legitimate owner of this session ran this" from "an attacker riding this session ran this" — the log looks identical either way.

**Note on 4688 in this scenario:** by default, Windows records that `net.exe`/`net1.exe` was spawned (typically by `powershell.exe` as the parent) but does **not** log the full command line unless command-line process auditing is explicitly enabled. Without that setting turned on, an analyst would see that `net.exe` ran, but not that it specifically ran `hacker /add` — a real visibility gap worth knowing about before relying on 4688 alone for this kind of investigation.

---

## Detection Finding: Logs Attribute Actions to Accounts, Not People

This is the core finding from the exercise, and it reframes how 4720/4732 should be read in a real investigation.

**The problem:** a Security log event showing "Maramba created an account" doesn't actually prove Maramba did it — it proves Maramba's *authenticated session* did it. If an attacker had compromised that session (stolen credentials, hijacked an active logon, or similar), the resulting 4720/4732 events would be **byte-for-byte identical** to this legitimate test. There is no field in the event that distinguishes "the real user, intentionally" from "an attacker, inside the real user's session."

**Why this matters for detection — it's not about the event, it's about context around it:**

| Question | Why it matters |
|---|---|
| **Who ran it?** | A helpdesk account running `net user /add` + `net localgroup administrators /add` is a role mismatch — helpdesk accounts have no legitimate reason to create admin accounts. That mismatch is the actual signal, not the command itself. |
| **When?** | A 3 AM account-creation event from a normally-9-to-5 account is far more suspicious than the same event during business hours. |
| **What ran immediately before it?** | Did this `net.exe` call come from a normal interactive session, or was it spawned by something unexpected — a macro-enabled document, a suspicious parent process? |
| **Is this normal for this specific account/machine?** | Routine on a domain controller managed by IT; a serious anomaly on a single end-user laptop like the one this test ran on. |

**The actual pivot, when the account itself looks wrong:** match the event's **Logon ID** back to the **4624** that opened that session in the first place. The Logon Type on that 4624 tells you *how* the session began — Type 2 (physical, someone at the keyboard) reads very differently from Type 10 (RDP, especially from an unfamiliar external IP). The account-creation event alone can't tell you which of those happened; the session's origin can.

---

## MITRE ATT&CK Mapping

| Technique | ID | Notes |
|---|---|---|
| Create Account: Local Account | T1136.001 | `net user hacker ... /add` |
| Account Manipulation | T1098 | `net localgroup administrators hacker /add` |

---

## Lab Evidence Captured
- Baseline `net user` output before the attack
- `net user hacker Password234 /add` command and success output
- Failed first `net localgroup` attempt (System error 1376) and the successful retry
- Confirmed `hacker` present in post-attack `net user` output, with admin rights
- Security log entries: 4720 (creation), 4732 (group add), 4726 (deletion on cleanup)
- Cleanup commands and final confirmation that the account was fully removed

---

## Why This Attack Is Different From Attacks 1 and 2

Attacks 1 and 2 both involved exploiting something — an exposed service, a weak credential. This attack exploited **nothing**. Every command used was a legitimate, signed, expected piece of the Windows operating system, run exactly as designed. The only thing that made it an attack rather than normal admin work was *intent and context* — which is precisely why Living Off the Land techniques are harder to catch with signature-based tooling than malware is, and why detection has to shift from "is this file bad?" to "does this activity, from this account, at this time, make sense?"

---

## Next Steps
- Enable command-line process auditing in a future lab iteration to confirm the 4688 visibility gap described above — currently only described from documentation, not directly observed
- Extend this exercise to a domain-joined environment once the Azure AD/Sentinel lab is built, to test the same technique against a helpdesk-tier account for a genuine role-mismatch detection scenario
- Write a basic Sigma rule for "4720 + 4732 for the same account within a short time window" as a detection rule, alongside the rules already scoped for Attacks 1 and 2
