# Active-Directory-Office-Lab

# Windows Server 2022 & Windows 11 Active Directory Mini-Office Lab

## Executive Summary
This project demonstrates the design, deployment, and administration of an enterprise-style Active Directory Domain Services (AD DS) environment using VirtualBox. 

The goal of this lab was to simulate a small office IT infrastructure from scratch—configuring network services (DNS, DHCP), implementing Organizational Units (OUs), enforcing Group Policies (GPOs), establishing secured network file shares, and executing Tier 1/2 Helpdesk administrative tasks.

---

## Network Topology & Lab Specifications

- **Hypervisor:** Oracle VM VirtualBox
- **Network Mode:** Isolated Internal Virtual Network (`intnet`)
- **Domain Controller (DC-01):** Windows Server 2022 Standard (`10.0.2.10/24`)
- **Client Workstation (WIN10-CLI):** Windows 11 Enterprise (DHCP / Static `10.0.2.50/24`)
- **Domain Name:** `lab.local`
