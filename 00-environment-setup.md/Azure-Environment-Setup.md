# Lab 00: Cloud Environment Setup & Architecture

## 1. Objective
I have to establish a secure, isolated cloud lab environment in Microsoft Azure to host an Active Directory domain, client workstations, and log analysis pipelines.

## 2. Architecture & Topology
* **Resource Group:** `rg-ad-lab-sec`
* **Region:** Central Poland
* **Virtual Network (VNet):** `vnet-ad-lab` (`10.0.0.0/16`)

### Subnet Breakdown
| Subnet Name | CIDR Block | Purpose |
| :--- | :--- | :--- |
| `snet-dc` | `10.0.1.0/24` | Domain Controller (Central Server) |
| `snet-client` | `10.0.2.0/24` | Domain-joined Workstations (Our Machines and Users)|
| `AzureBastionSubnet` | `10.0.3.0/26` | Secure RDP/SSH Management Access |

## 3. Component Specifications
* **Cloud Platform:** Azure for Students
* **Domain Controller (DC):** 
  * OS: Windows Server 2022 Datacenter
  * Private IP: 10.0.1.0/24
* **Client Workstation:** 
  * OS: Windows 10 Pro
  * Private IP: 10.0.2.0/24

## 4. Implementation Steps
1. **Subscription & Resource Group:** Provisioned via Azure for Students credits and deployed `rg-ad-lab-sec`.
2. **VNet Configuration:** Created `vnet-ad-lab` with custom address space and segmented subnets.

## Regions, Zones, and VM Size Availability

My subscription (Azure for Students) is restricted to a limited set of regions —
Austria East failed outright due to this restriction.

Switching to North Italy fixed the region problem but introduced a new one:
no VM sizes appeared as available at all. The actual cause wasn't the region —
it was the "Security Type" setting, which defaults to "Trusted launch virtual
machines." Several free-tier/burstable sizes (B1s, B2ats_v2, B2pts_v2) don't
support Trusted launch, so the portal silently showed zero sizes even though
sizes existed for my subscription. Switching Security Type to "Standard" was
the actual fix — not the region.

From there, I found Poland Central supported the B2ats_v2 size I needed at
Standard security type, and used that as my final region.

### Second issue: zonal allocation failure

With the region and size resolved, VM deployment still failed:

    Allocation failed. We do not have sufficient capacity for the requested
    VM size in this zone. (Code: ZonalAllocationFailed)

Cause: the VM was pinned to a specific availability zone (Zone 1) under
"Availability options," and that specific zone didn't have capacity for
B2ats_v2 at that moment — a transient, zone-specific shortage, not a
subscription-wide issue.

Fix: changed Availability options to "No infrastructure redundancy required,"
which lets Azure place the VM in any zone with available capacity instead of
locking it to one. Deployment succeeded immediately after.


## Promoting the VM to a Domain Controller 
Azure VM NICs are dynamin by default and of which we need the Domain Controller 
to have a static IP address , this is for stabilization and also so that the Domain 
Controller points to its DNS settings . 

I had to install the AD DS role , I did this through RDP using the VM's public IP into my VM which was my first time. I then got into a Windows Desktop and ran the `Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools` command running as 
Administrator. I then promoted the server to a Domain Controller , picked a lab-only 
domain which I named adlab.local and set the password.

I then had to ensure that DC promotion succeeded , I had to run the `dcdiag` in which it returned 'passed test' on every line especially for DNS, Advertising and NetLogon. 
We want to see 'passed test' specifically on DNS because Active Directory relies heavily on DNS to locate the Domain Controllers and services . For Advertising , it is important because it checks whether the DC is correctly advertising itself as a Domain Controller through Active Diretory's DC locator mechanism and last but not least , Netlogon . The Netlogon is a windows which supports secure channel operations , Domain authentication functions and DC locator registration .

## Creating an Organizational Unit(OU)
I then had to create an OU in which to organize our Domain user's into structure. This is to apply GPO's to them .I created 3 user accounts with their associated passwords which were quite simple , this is for future purposes - our password spray attack . 

## Installing and Configuring Sysmon 
We have to install Sysmon on our VM because it helps us record the system activity which is happening on our server which is the Domain Controller such as for example Process creation and command line arguments , network connections initiated by processes . We then have to also download the SwiftOnSecurity configuration file which tells Sysmon what to monitor , what to include and what to exclude because without this file , enabling everything can generate huge amounts of noise. This was the command I ran after installing the two through GUI , `   .\Sysmon64.exe -accepteula -i sysmonconfig-export.xml` and ran `dcdiag` to check if everything is running perfectly . This was all done on the Windows VM . 

## Sysmon and Windows Security events channels 
I had to create a storage layer in which Sentinel sits on top off. This was done through making a Log Analytics workspace `law-ad-space` in Azure portal then enabled Microsoft Sentinel in that workspace of I had created - This is to turn the plain logs in our storage layer into an actual SIEM with analytics/alerting capabilities. I then had to connect the standard Windows Security event via AMA(Azure Monitor Agent) because it is responsible for collecting certain type of logs and deciding where those logs go to based on the DCR(Data Collection Rule) we define. Had to select our DC as the target resource because it is the one which will be monitoring for attacks against our AD. 

I had a small hiccup that I encountered where the AMA only pulls the standard security events but it does not automatically pull Sysmon events because Sysmon has it dedicated log channel `Microsoft-Windows-Sysmon/Operational`. So I then had to create a second custom DCR to manually specify the Xpath/channel `Microsoft-Windows-Sysmon/Operational` targeting the same DC sending to the same workspace where our Sentinel is located. 

Now I had to verify both log types are actually arriving , had to run these two separate commands in the KQL query window : `SecurityEvent | take 10` and `Event | where Source == "Microsoft-Windows-Sysmon" | take 10` where they both returned logs . 
