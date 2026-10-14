# Lab 00: Cloud Environment Setup & Architecture

## 1. Objective
Establish a secure, isolated cloud lab environment in Microsoft Azure to host an Active Directory domain, client workstations, and log analysis pipelines.

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
  * OS: Windows Server [2022/2019 Datacenter]
  * Private IP: [10.0.1.0/24]
* **Client Workstation:** 
  * OS: Windows [10/11 Pro or Enterprise]
  * Private IP: [10.0.2.0/24]

## 4. Implementation Steps
1. **Subscription & Resource Group:** Provisioned via Azure for Students credits and deployed `rg-ad-lab-sec`.
2. **VNet Configuration:** Created `vnet-ad-lab` with custom address space and segmented subnets.
3. **Connectivity Verification:** Tested internal routing and subnet isolation.

## 5. Next Steps
* Deploy and configure Active Directory Domain Services (AD DS) and DNS on the Domain Controller.

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
