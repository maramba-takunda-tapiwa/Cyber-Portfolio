# Cyber-Portfolio

Second-year Computer Science student (University of Debrecen) building 
toward a SOC Analyst role, with a longer-term goal of Junior Penetration 
Tester. This repo documents hands-on lab work and attack/detection 
exercises as I work through them.

## Start here
- [AD/SIEM Detection Lab — Azure Environment Setup](ad-siem-detection-lab-project/01-Azure-Environment-Setup.md)
- [AD/SIEM Detection Lab — Log Forwarding to Sentinel](ad-siem-detection-lab-project/02-log-forwarding-sentinel.md)
- [AD/SIEM Detection Lab — Kerberos Password Spray Detection](ad-siem-detection-lab-project/03-spray-detection-kerbrute.md)
- [SOC Detection Lab — Attack 1: Port Scan & Bindshell Exploitation](soc-detection-lab-project/attack-1-bindshell.md)
- [SOC Detection Lab — Attack 2: SSH Credential Brute Force](soc-detection-lab-project/attack-2-ssh-bruteforce.md)
- [SOC Detection Lab — Attack 3: Living off the Land — Backdoor Admin Account](soc-detection-lab-project/attack-3-lotl-backdoor-account.md)
- [Home Lab Setup & Troubleshooting](00-home-lab/lab-setup-troubleshooting.md)

## Skills demonstrated
Nmap reconnaissance · Hydra & Kerbrute credential attacks · Active Directory 
administration · Microsoft Sentinel & KQL · Sysmon configuration · 
Linux/Windows log analysis · MITRE ATT&CK mapping · Azure networking · 
VirtualBox lab administration · cross-system timestamp correlation · 
Wireshark packet analysis

## Repo structure
- `ad-siem-detection-lab-project/` — Azure-hosted Active Directory + Microsoft 
  Sentinel SIEM build, with a Kerberos password spray attack and detection
- `soc-detection-lab-project/` — attack + detection writeups against a 
  Kali/Metasploitable2 home lab
- `00-home-lab/` — initial lab environment setup and troubleshooting log
