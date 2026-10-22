# AD/SIEM Detection Lab — Password Spray via Kerberos Pre-Authentication

## Overview

With the Domain Controller built, Sysmon installed, and both standard Security events and Sysmon logs confirmed flowing into Microsoft Sentinel, this is the actual attack phase of the project: a Kerberos pre-authentication password spray against my own lab AD domain (`adlab.local`), played from both sides — attacker and defender — same as the three Metasploitable attacks, but this time against a real Active Directory environment I built myself in Azure.

---

## Deploying the Attacker VM

I deployed a small Ubuntu VM (`ub-vm`, B1s size) to act as the attacking machine, placed deliberately inside the same virtual network as the Domain Controller rather than attacking from the open internet — this keeps the traffic internal, which is both safer and more realistic for how a password spray actually plays out in a real breach (an attacker who already has a foothold inside the network, not someone blasting Kerberos at a public IP).

### Networking issue #1: wrong VNet entirely

The first time I deployed the attacker VM, it landed in a completely different, auto-generated virtual network than my Domain Controller, even though I thought I'd pointed it at the same one. VNets in Azure are fully isolated from each other by default, so no amount of correct IP addressing was ever going to let the two VMs talk — they were sitting on two separate networks that didn't know about each other at all. I deleted and redeployed the attacker VM, this time explicitly selecting the DC's existing VNet and subnet from the dropdown rather than trusting whatever the wizard defaulted to.

### Networking issue #2: the same VNet mismatch, in disguise

After "fixing" it, I ran into what looked like an IP conflict — the attacker VM and the DC both appeared to have the identical private IP address. I checked both VMs' actual Networking tabs side by side in the Azure portal, and it turned out this wasn't a true conflict at all: the two VMs were in two different, separately auto-named VNets (`vnet-polandcentral-1` vs `vnet-polandcentral-2`) that happened to independently assign the same address, since neither VNet was aware of the other. Same root cause as the first issue — the deployment wizard hadn't actually used the existing VNet — just presenting differently. I redeployed a second time, this time explicitly confirming on the final review screen (not just the earlier tab) that the VNet matched exactly.

### DNS resolution

Once both VMs were confirmed on the same VNet, I still couldn't resolve `adlab.local` using the system's default resolver (`127.0.0.53`) — it returned `REFUSED`. Manually pointing `nslookup` at the DC's IP directly worked, which told me DNS itself was functioning correctly on the DC, but the attacker VM's default DNS wasn't pointed at it. I confirmed this with an SRV record lookup specifically:

```
nslookup -type=SRV _ldap._tcp.dc._msdcs.adlab.local 172.16.0.4

```
<img width="771" height="157" alt="image" src="https://github.com/user-attachments/assets/e2fc1cd7-9cdb-4ea4-bc3b-f580f25896f6" />

This returned a clean answer, pointing to `vm-ad-dc.adlab.local` — the same mechanism real AD clients use to locate a Domain Controller, and the same thing `dcdiag`'s Advertising test checks internally. That confirmed DNS, the VNet fix, and the DC's own service advertisement were all genuinely working together.

---

## Detection Hypothesis — Written Before the Attack

Before touching Kerbrute, I wrote down what I expected to see, same discipline as every attack before this one.

A password **spray** has a different shape from Attack 2's SSH **brute force**: instead of many password attempts against one account, a spray tests few attempts (often just one) against **many** accounts. I expected:

- **Event ID 4768** (Kerberos TGT requested) for any successful authentication
- **Event ID 4771** (Kerberos pre-authentication failed) for wrong-password attempts against real accounts
- The pattern should show attempts spread across all target accounts, from one source IP, in a tight time window — the opposite shape from Attack 2's single-account brute force

---

## Installing Kerbrute

Downloaded the static binary directly on the attacker VM:

```bash
wget https://github.com/ropnop/kerbrute/releases/latest/download/kerbrute_linux_amd64
chmod +x kerbrute_linux_amd64
```

No package manager install needed — single self-contained binary.

---

## Building the Username List (and an Accidental Finding)

I built `accounts.txt` from the real test accounts in my `Lab-Users` OU, but also deliberately threw in a handful of invalid, made-up usernames out of curiosity, to see how the Domain Controller would respond to accounts that don't actually exist.
<img width="415" height="215" alt="image" src="https://github.com/user-attachments/assets/e1239050-b520-48b8-b55f-5f575d3b0013" />


---

## Running the Spray

```bash
./kerbrute_linux_amd64 passwordspray -d adlab.local --dc 172.16.0.4 accounts.txt 'Summer2026!'
```

**Result:**
<img width="757" height="307" alt="image" src="https://github.com/user-attachments/assets/1fe4a7c6-582e-43f9-bb79-aec647d966d8" />

I also ran a second pass with a wrong password, to generate the failure side of the evidence:

```bash
./kerbrute_linux_amd64 passwordspray -d adlab.local --dc 172.16.0.4 accounts.txt 'Summer2046!'
```

**Result:**
<img width="753" height="266" alt="image" src="https://github.com/user-attachments/assets/768ca09b-6bb3-4e18-a8b9-142462ad50d0" />


(The `--dc` flag was necessary because Kerbrute's own internal KDC discovery uses the system's default resolver, which wasn't pointed at the DC — same underlying DNS issue as above, worked around rather than fully fixed at the OS level for this session.)

---

## Pulling the Evidence — With a Few Real Detours

### First attempt: filtering by account name, nothing useful came back

My first instinct was to filter `SecurityEvent` for rows where `Account` contained "cate" or "maramba." This returned results, but they turned out to be my own RDP/admin activity on the DC (`ADLAB\maramba`), not the spray — a false positive caused by my own account name being a substring match for the test accounts' names too.
<img width="901" height="377" alt="image" src="https://github.com/user-attachments/assets/2addf8c1-9dee-4d45-bc64-ceecc81dd0be" />


### Second attempt: filtering by the real Kerberos event IDs and source IP

I narrowed the query to the actual Kerberos event IDs instead of guessing at account name substrings:

```kql
SecurityEvent
| where TimeGenerated > ago(1h)
| where EventID in (4768, 4771)
| project TimeGenerated, EventID, Activity, IpAddress
```
<img width="882" height="413" alt="image" src="https://github.com/user-attachments/assets/412c6a65-b254-4d69-aa60-5bcb0b248144" />


This returned real hits, and critically, the `IpAddress` field matched my attacker VM's actual IP — that correlation, not the account name, is what confirmed these were genuinely my spray's events and not unrelated DC background noise.

### Third detour: the Account field was empty

The generic `Account` column in `SecurityEvent` didn't populate for these specific 4768 events. Expanding a raw event showed the real field holding the username was `TargetUserName`, not `Account` — a field-mapping quirk specific to this event type rather than missing data. Updated the query:

```kql
SecurityEvent
| where TimeGenerated > ago(1h)
| where EventID == 4771 or EventID == 4768
| project TimeGenerated, EventID, Activity, IpAddress, TargetUserName
```

---

## Evidence — Final Results

This query returned exactly what the spray should have produced, plus more:

| TargetUserName | EventID | Meaning |
|---|---|---|
| k.cate | 4768 | Successful TGT (correct password) |
| maramba.s | 4768 | Successful TGT (correct password) |
| t.maramba | 4771 | Pre-auth failed (wrong password, real account) |
<img width="903" height="330" alt="image" src="https://github.com/user-attachments/assets/cdad0b86-cf3e-4de4-b2f2-8f9ac491687d" />


---

## Finding: Hypothesis vs. Reality

My original hypothesis was correct on the core shape — attempts spread across accounts, from one source, in a tight window, matching the 4768/4771 split I predicted. Where it fell short: I hadn't accounted for what would happen with usernames that don't actually exist in the directory, because I hadn't planned to test that at all — it came from curiosity about the wordlist, not the original plan.

## Finding: Kerberos Username Enumeration (Accidental Discovery)

Several of the invalid, made-up usernames I'd mixed into `accounts.txt` out of curiosity also generated **Event 4768 (TGT requested)** entries in the logs — the same event type as a genuine successful login. This pointed me to a real, named attacker technique, separate from the password spray itself: **Kerberos pre-authentication username enumeration.**

The mechanism: when a Kerberos AS-REQ is sent for a username that doesn't exist, the KDC responds with `KRB5KDC_ERR_C_PRINCIPAL_UNKNOWN`. When the username *does* exist, the KDC instead responds with `KRB5KDC_ERR_PREAUTH_REQUIRED` (asking for the password) — meaning **the server's response alone reveals whether an account exists, before any password is ever tested.** This is exactly why Kerbrute ships a dedicated separate mode, `userenum`, built specifically for this — real attackers use it first to build a clean list of valid usernames, since it generates no login failures and triggers no account lockouts.

I hadn't run `userenum` — I ran `passwordspray` with a mixed valid/invalid username list, which demonstrated the same underlying mechanism as a side effect rather than by design. Worth being precise about that distinction rather than overclaiming I'd specifically tested the enumeration technique on purpose.

---

## MITRE ATT&CK Mapping

| Technique | ID | Notes |
|---|---|---|
| Password Spraying | T1110.003 | Kerbrute `passwordspray` against `adlab.local` |
| Valid Accounts | T1078 | Successful authentication as `k.cate` and `maramba.s` |

*(Kerberos username enumeration via AS-REQ response differences is a reconnaissance-adjacent technique closely tied to password spraying preparation, though I haven't pinned it to a specific standalone ATT&CK sub-technique ID — worth researching further before stating one definitively.)*

---

## Lab Evidence Captured
- `dcdiag` output confirming DC health (Advertising, NetLogons, partition tests passing)
- SRV record lookup confirming DNS correctly advertises the DC
- Kerbrute terminal output for both the successful and failed password runs
- Sentinel KQL query results: 4768/4771 events, correlated by source IP to the attacker VM
- `TargetUserName` field showing both real test accounts and the invalid usernames that still triggered Kerberos responses

---

