# Lab 00: Cloud Environment Setup & Architecture

## 1. Objective
Establish a secure, isolated cloud lab environment in Microsoft Azure to host an Active Directory domain, client workstations, and log analysis pipelines.

## 2. Architecture & Topology
* **Resource Group:** `rg-ad-lab-sec`
* **Region:** []
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
