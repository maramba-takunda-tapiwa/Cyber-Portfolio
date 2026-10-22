# Lab 00: Cloud Environment Setup & Architecture

## 1. Objective
I have to establish a secure, isolated cloud lab environment in Microsoft Azure to host an Active Directory domain, client workstations, and log analysis pipelines.

## 2. Architecture & Topology
* **Resource Group:** `rg-ad-lab-sec`
* **Region:** Poland Central
* **Virtual Network (VNet):** `vn-ad` (`10.0.0.0/16`)

### Subnet Breakdown
| Subnet Name | CIDR Block | Purpose |
| :--- | :--- | :--- |
| `snet-dc` | `10.0.1.0/24` | Domain Controller (Central Server) |
| `snet-workstation` | `10.0.2.0/24` | Domain-joined Workstations (Our Machines and Users)|


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
2. **VNet Configuration:** Created `vn-ad` with custom address space and segmented subnets.

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
<img width="512" height="47" alt="image" src="https://github.com/user-attachments/assets/53f530b1-1a98-4f8b-b291-85997fc6250a" />
<img width="515" height="33" alt="image" src="https://github.com/user-attachments/assets/dd5aba6e-b632-43c4-ab2b-41383d7a30cb" />
<img width="770" height="767" alt="image" src="https://github.com/user-attachments/assets/3462d184-78b0-44e1-bd12-447601f2064b" />




## Creating an Organizational Unit(OU)
I then had to create an OU in which to organize our Domain user's into structure. This is to apply GPO's to them .I created 3 user accounts with their associated passwords which were quite simple under the group name called `Lab-Users` , The password were relatively weak but did not break the password policy which enforces password complexity and also for the password-spray that we are going to do later on.
<img width="721" height="293" alt="image" src="https://github.com/user-attachments/assets/74936a25-7794-4ba0-830f-6998cbfa74f2" />






