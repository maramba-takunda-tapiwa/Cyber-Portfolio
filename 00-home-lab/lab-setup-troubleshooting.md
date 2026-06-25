# Home Lab Setup — Kali to Metasploitable2 Connectivity Troubleshooting

## Summary

Set up a home lab (Kali Linux as attacker machine, Metasploitable2 as vulnerable target) in VirtualBox. Initial connectivity between the two VMs failed repeatedly. This document tracks the actual troubleshooting process — including the false leads — because the process matters more than the fix.

---

## Environment

- **Attacker:** Kali Linux (VM name: Hack_Marr)
- **Target:** Metasploitable2
- **Hypervisor:** Oracle VirtualBox

---

## Symptom

```bash
ping 192.168.56.101
```
Result: `Destination Host Unreachable` on every packet, 100% packet loss.

```bash
arp -a
? (192.168.56.101) at <incomplete> on eth0
```
Kali could not resolve Metasploitable's MAC address at all — meaning the failure was happening at Layer 2 (Data Link), not just at the ICMP/ping level.

---

## Hypotheses Tested (in order)

### 1. OS type mismatch (Debian vs Oracle Linux label)
**Hypothesis:** Mismatched "Operating System" labels in VirtualBox settings might block networking between the two VMs.
**Result: Ruled out.** OS type labels in VirtualBox are cosmetic (icon/default hints only) and have zero effect on actual network communication. Networking protocols are OS-agnostic — this was never a real lead.

### 2. Metasploitable kernel panic on boot
**Separate issue, found along the way:** Metasploitable crashed on boot with `Kernel panic - not syncing: IO-APIC + timer doesn't work!` — a known compatibility issue between the old Metasploitable2 kernel (built ~2012) and modern VirtualBox/host CPU virtualization defaults.
**Fix:** Settings → System → Motherboard → unchecked "Enable I/O APIC", set Processor count to 1.
**Result:** Metasploitable booted successfully to login. This fixed the boot problem but did NOT fix the network connectivity problem — they were two separate issues running in parallel.

### 3. Host-only adapter mismatch between VMs
**Hypothesis:** Kali and Metasploitable might be attached to two different Host-Only Adapter instances (VirtualBox can have multiple), which would put them on separate isolated networks despite both showing `192.168.56.x` addresses.
**Result: Ruled out.** Checked Settings → Network → Adapter 1 on both VMs. Both showed identical configuration: Attached to "Host-only Adapter", Name: "VirtualBox Host-Only Ethernet Adapter", same adapter type, both with Virtual Cable Connected. Confirmed identical — not the cause.

### 4. Switched to Internal Network, attempted manual static IPs
**Action taken:** Switched both VMs from Host-only Adapter to Internal Network (which has no built-in DHCP — requires fully manual IP assignment).
**Result:** Manually set Metasploitable's IP via `ifconfig` — worked correctly (`192.168.10.10` applied cleanly). Attempted the same on Kali via `ifconfig eth0 192.168.10.20` — command appeared to run but ping still failed afterward with the same `<incomplete>` ARP result.

### 5. Root cause — DHCP failure on Kali during/after install
**What actually happened:** During a Kali reinstall (triggered out of frustration, not because the original install was confirmed broken), the installer flagged a DHCP-related network configuration warning, which was skipped to continue installation. This meant Kali was never reliably getting a valid IP via DHCP on the host-only network — explaining the persistent Layer 2 resolution failure (`<incomplete>` ARP) seen throughout every earlier attempt.

**Actual fix:** Inside Kali → Advanced Network Configuration → Ethernet adapter → IPv4 tab → disabled "Automatic (DHCP)" → set Manual → assigned static IP `192.168.56.105/24`.

**Result:** Ping to Metasploitable succeeded immediately after applying the static IP.

---

## Actual Root Cause

DHCP was not reliably assigning Kali a valid IP address on the host-only network (likely due to the DHCP warning seen during install). Without a valid IP, Kali could not be reached at Layer 2, regardless of how the adapters were named or attached — explaining why ARP resolution failed consistently across multiple earlier attempts (Host-only, then Internal Network) even when the adapter-level configuration looked correct.

---

## Honest Lesson Learned

**The Kali reinstall was not actually necessary.** The fix — disabling DHCP and assigning a static IP manually — could have been applied to the original Kali installation the moment DHCP was suspected as the problem. Rebuilding the entire VM from scratch cost significant time for a fix that takes under a minute once you know where to look.

**Going forward:** when a VM shows no/invalid IP or ARP `<incomplete>` results, check DHCP status and try a manual static IP **before** assuming the VM itself is broken or reinstalling. Reinstalling should be a last resort, not an early troubleshooting step.

**Why this matters for SOC work:** this is a real example of root cause analysis under frustration — multiple hypotheses tested, several ruled out, one dead end (the reinstall) before reaching the actual cause. That iterative process, not the final fix, is the actual transferable skill.

---

## Diagnostic Commands Used

| Command | Purpose |
|---|---|
| `ifconfig` / `ip a` | Check interface name, assigned IP, subnet |
| `ip route` | Confirm routing table has a path to the target subnet |
| `arp -a` | Check Layer 2 resolution status — `<incomplete>` means no MAC ever returned |
| `ping <ip>` | Test end-to-end reachability |
| `sudo ifconfig eth0 <ip> netmask <mask> up` | Manually assign IP live (does not persist on reboot) |

---

## Current Lab State

- Kali: static IP `192.168.56.105/24`, Host-only Adapter
- Metasploitable2: `192.168.56.101/24`, Host-only Adapter, IO-APIC disabled, 1 CPU
- Connectivity confirmed working via sustained ping test
- Ready for Phase 1 of the SOC detection lab project (attack simulation + log/packet capture + analyst write-up)
