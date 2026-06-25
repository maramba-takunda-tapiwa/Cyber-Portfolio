# DNS Deep Dive — SOC Analyst Notes

## How DNS Actually Works

```
You type: google.com
   ↓
Your device asks: Recursive Resolver (usually your ISP or 8.8.8.8)
   ↓
Resolver asks: Root DNS Server → "who handles .com?"
   ↓
Root replies: TLD server for .com
   ↓
TLD server replies: "ask google.com's authoritative server"
   ↓
Authoritative server replies: "google.com = 142.250.x.x"
   ↓
Resolver caches and returns the IP to you
```

This entire process normally takes milliseconds and happens over **UDP port 53** (switches to TCP only for larger responses, e.g. DNSSEC or zone transfers).

---

## DNS Record Types

| Record | Purpose |
|---|---|
| **A** | Maps a domain to an IPv4 address |
| **AAAA** | Maps a domain to an IPv6 address |
| **MX** | Mail server for the domain |
| **TXT** | Arbitrary text — often used for SPF/DKIM/DMARC email verification |
| **NS** | Which name servers are authoritative for the domain |
| **CNAME** | Alias — points one domain to another domain |
| **PTR** | Reverse lookup — IP to domain name |

---

## DNS Attacks — Full List

### 1. DNS Tunnelling
Encoding stolen data into subdomains, exfiltrating via DNS queries because DNS is rarely blocked. The long, random-looking subdomain isn't noise — it's literally the encoded stolen data being smuggled out chunk by chunk, query by query.

### 2. DNS Spoofing / Cache Poisoning
Attacker injects fake DNS records into a resolver's cache, so when victims query a legitimate domain, they get redirected to a malicious IP instead.
- **Detection:** Unexpected IP for a well-known domain, TTL anomalies, mismatched DNS responses across different resolvers

### 3. DNS Hijacking
Attacker compromises the actual DNS settings (router, registrar account, or resolver) to redirect ALL queries through a malicious server.
- **Detection:** DNS settings changed without authorization, all users on a network suddenly resolving to wrong IPs

### 4. Fast Flux DNS
Attackers rapidly rotate the IP addresses behind a malicious domain (sometimes every few minutes) to evade IP-based blocklists — common in botnet C2 infrastructure.
- **Detection:** Domain resolving to dozens of different IPs in a short time window, very low TTL values

### 5. DGA — Domain Generation Algorithms
Malware that algorithmically generates hundreds of random-looking domains per day so C2 infrastructure can't be blocked by blocklisting a single domain.
- **Detection:** High volume of NXDOMAIN (non-existent domain) responses, randomly-generated-looking domain names with no real meaning (e.g. `xqzplv8f.com`)

### 6. Typosquatting
Registering domains that are common misspellings of legitimate ones (`gooogle.com`, `paypa1.com`) to catch users who mistype or to use in phishing links.
- **Detection:** Often caught via threat intel feeds rather than logs directly — but DNS logs showing internal users querying suspiciously-similar-to-legitimate domains is a red flag

---

## Normal vs Suspicious DNS in Logs

**Normal:**
```
Query: www.google.com → A record → 142.250.190.78
Query: outlook.office365.com → A record → 52.96.x.x
```
Few queries per session, recognizable domains, normal TTLs.

**Suspicious:**
```
Query: 7f3a9b2c1d8e5f4a6b9c0d1e2f3a4b5c.evil-domain.com
Query: k8p2m9x.evil-domain.com
Query: q4r7t1z.evil-domain.com
```
Random-looking subdomains, repeated queries to the same parent domain, high frequency, often NXDOMAIN responses mixed in (DGA malware querying domains that don't even exist yet because the C2 hasn't registered today's domain).

---

## Self-Test — Scenarios

### Q1: Single internal host querying 200 different random-character domains per minute, mostly NXDOMAIN responses. What's happening?

**My answer (correct):** Domain Generation Algorithm (DGA) — malware generating multiple domains per day so C2 infrastructure can't be blocklisted by targeting a single domain. Classic botnet/C2 behavior.

### Q2: User reports banking site "looks slightly different," asks to re-enter password. DNS resolution for the bank's domain returns an IP that doesn't match any known legitimate IP. What attack, and what's the difference between this happening at the resolver level vs the user's local machine?

**My first answer:** Correctly identified this as DNS spoofing/cache poisoning at the resolver level. Initially missed the local-machine half of the comparison.

**Full answer (corrected):**

- **Resolver-level poisoning:** The cache on a shared DNS server (ISP resolver or company internal DNS) gets corrupted with a fake IP for the bank's domain. **Everyone** querying that resolver gets redirected — potentially hundreds/thousands of victims from one poisoned resolver.

- **Local machine-level (hosts file):** Every computer has a hosts file (`C:\Windows\System32\drivers\etc\hosts` on Windows, `/etc/hosts` on Linux/Mac) that maps domains to IPs manually, and is checked **before** any DNS query goes out. If malware edits this file directly, only **that one machine** is redirected — no DNS server touched, no external cache poisoned, nobody else affected.

- **Why the distinction matters for incident scope:** Same symptom (wrong IP for a real domain) but wildly different response scale.
  - Resolver-level → huge investigation scope: how many users hit that resolver, is it internal or upstream ISP, may need network-wide DNS cache flush and broad alert.
  - Hosts file-level → narrow scope: single-endpoint forensics, identify the malware that got local admin access to edit a protected system file, and trace how it got there.

**My initial wrong guess:** confused local-machine poisoning with botnet/zombie recruitment — that's a different concept entirely (a device being remotely controlled as part of a botnet, not DNS resolution being tampered with locally). Don't conflate the two.

**Tier 1 response sequence for resolver-level poisoning (the real workflow, not just "check if false positive"):**
1. **Verify independently** — query the same domain against a trusted external resolver (8.8.8.8, 1.1.1.1) and compare results. Mismatch = confirmed poisoning, not a false positive.
2. **Check scope** — is it just this one user, or are others on the network reporting the same symptom? Tells you resolver-wide vs isolated.
3. **Escalate per playbook** — confirmed resolver-level poisoning is an active, spreading incident; goes to Tier 2/network team immediately, not a "monitor and wait" situation.
4. **Advise the user in the meantime** — don't enter the password, avoid that resolver, use mobile data temporarily for anything sensitive.

**Key lesson:** "Check if it's a false positive" needs a concrete method behind it, not just the instinct. The method here is cross-referencing against a trusted external resolver.

### Q3: What port does DNS run on, what protocol by default, and when does it switch?

**My answer (correct):** Port 53, UDP by default, switches to TCP when responses are larger (e.g. DNSSEC, zone transfers).

---

## Key Takeaways

- DNS poisoning can happen at vastly different scales (resolver vs local) with identical symptoms to the end user — scope determination is the analyst's job, not assumption
- DGA and fast flux both exist to defeat blocklisting — they're evasion techniques, not the attack itself
- Verification before escalation always needs a concrete method (e.g. cross-resolver check), not just "double-check it"
