# 01 — Domain Design

## Overview

This document covers the Active Directory domain design for the MedNet Enterprise Lab, including the domain configuration, OU structure, naming conventions, and user account model. The design simulates a realistic hospital IT environment to reflect real-world healthcare infrastructure standards.

---

## Domain Configuration

| Setting | Value |
|---|---|
| Domain Name | `mednet.lab` |
| Domain Controller | `dc01` |
| DC Role | Global Catalog |
| AD Site | Default-First-Site-Name |
| Forest/Domain Functional Level | Windows Server 2016+ |

---

## OU Structure

The domain uses a single top-level organizational OU (`MedNet`) to separate managed objects from default AD containers. All user accounts, computer objects, service accounts, and security groups are organized within this hierarchy.

```
mednet.lab
└── MedNet (top-level OU)
    ├── Departments
    │   ├── Administrative
    │   │   ├── Finance
    │   │   ├── HR
    │   │   └── Reception
    │   ├── Clinical
    │   │   ├── Nursing
    │   │   ├── Pharmacy
    │   │   └── Physicians
    │   └── IT
    │       ├── Helpdesk
    │       └── Systems
    ├── Admin Accounts
    ├── Security Groups
    ├── Service Accounts
    └── Workstations
        ├── Computers
        └── Servers
```

### OU Design Rationale

| OU | Purpose |
|---|---|
| `Departments` | Contains all standard user accounts, organized by role. Primary GPO link point for department-scoped policies. |
| `Admin Accounts` | Holds privileged accounts (Domain Admins, Helpdesk Admins) separate from standard users — enforces least-privilege separation. |
| `Security Groups` | Centralized location for all custom security groups used in RBAC and resource access delegation. |
| `Service Accounts` | Dedicated OU for application service accounts (Zabbix, Wazuh, osTicket). Prevents service accounts from being mixed with user objects. |
| `Workstations` | Holds domain-joined computer objects. Split into `Computers` (endpoints) and `Servers` to allow machine-level GPOs to be scoped appropriately. |

> **Note:** The `Workstations` parent OU was named as such because `Computers` is a reserved default container in Active Directory and cannot be reused as an OU name at the domain root level.

---

## Screenshots

### Full Domain Structure

The following screenshot shows the complete OU hierarchy as viewed in Active Directory Users and Computers, with all custom OUs visible alongside the default AD containers.

![AD OU Structure](../screenshots/01-domain-design.md_01.png)

### Populated Department OU — Clinical/Physicians

The following screenshot shows the `Physicians` sub-OU populated with representative user accounts, demonstrating how department OUs are used to organize users by role.

![AD Physicians OU](../screenshots/01-domain-design.md_02.png)

---

## Naming Conventions

### User Accounts

| Convention | Format | Example |
|---|---|---|
| Username | `firstname.lastname` | `s.mitchell` |
| Display Name | `Firstname Lastname` | `Sarah Mitchell` |
| Clinical titles | Included in display name only | `Dr. Sarah Mitchell` |

The `firstname.lastname` format is standard in healthcare IT environments and aligns with common UPN patterns (`s.mitchell@mednet.lab`).

### Representative User Accounts

The following test accounts are distributed across department OUs to simulate a realistic hospital staff structure:

| Display Name | Username | OU Path |
|---|---|---|
| Dr. Sarah Mitchell | s.mitchell | Departments/Clinical/Physicians |
| Dr. James Ortega | j.ortega | Departments/Clinical/Physicians |
| Lisa Nguyen | l.nguyen | Departments/Clinical/Nursing |
| Mark Evans | m.evans | Departments/Clinical/Pharmacy |
| Karen Booth | k.booth | Departments/Administrative/HR |
| Tom Reyes | t.reyes | Departments/Administrative/Finance |
| Dana Cole | d.cole | Departments/Administrative/Reception |
| Alex Turner | a.turner | Departments/IT/Helpdesk |

### Computer Objects

| Convention | Format | Example |
|---|---|---|
| Workstations | `WS-DEPT-##` | `WS-CLIN-01` |
| Servers | `SRV-ROLE-##` | `SRV-ZABBIX-01` |

Computer objects for Phase 3 endpoint VMs will follow this convention and be placed in the appropriate `Workstations/Computers` or `Workstations/Servers` sub-OU upon domain join.

### Service Accounts

| Convention | Format | Example |
|---|---|---|
| Service Accounts | `svc-appname` | `svc-zabbix` |

Service accounts are placed in the `Service Accounts` OU and follow a `svc-` prefix to distinguish them clearly from standard user accounts in logs and audit reports.

---

## Security Groups

All custom security groups are located in `OU=Security Groups,DC=mednet,DC=lab`. Groups follow a `Department-Role` naming convention and are used for RBAC across domain-joined resources including the Samba file server.

### Department Security Groups

| Group Name | Scope | Members | Purpose |
|---|---|---|---|
| `Clinical-Physicians` | Global / Security | s.mitchell, j.ortega | Access to `physicians` file share |
| `Clinical-Nursing` | Global / Security | l.nguyen | Access to `nursing` file share |
| `Clinical-Pharmacy` | Global / Security | m.evans | Access to `pharmacy` file share |
| `Admin-HR` | Global / Security | k.booth | Access to `hr` file share |
| `Admin-Finance` | Global / Security | t.reyes | Access to `finance` file share |
| `Admin-Reception` | Global / Security | d.cole | Access to `reception` file share |
| `IT-Staff` | Global / Security | a.turner | Access to `it` file share |

Groups were created using PowerShell on the domain controller:

```powershell
New-ADGroup -Name "Clinical-Physicians" -GroupScope Global -GroupCategory Security -Path "OU=Security Groups,DC=mednet,DC=lab"
New-ADGroup -Name "Clinical-Nursing" -GroupScope Global -GroupCategory Security -Path "OU=Security Groups,DC=mednet,DC=lab"
New-ADGroup -Name "Clinical-Pharmacy" -GroupScope Global -GroupCategory Security -Path "OU=Security Groups,DC=mednet,DC=lab"
New-ADGroup -Name "Admin-HR" -GroupScope Global -GroupCategory Security -Path "OU=Security Groups,DC=mednet,DC=lab"
New-ADGroup -Name "Admin-Finance" -GroupScope Global -GroupCategory Security -Path "OU=Security Groups,DC=mednet,DC=lab"
New-ADGroup -Name "Admin-Reception" -GroupScope Global -GroupCategory Security -Path "OU=Security Groups,DC=mednet,DC=lab"
New-ADGroup -Name "IT-Staff" -GroupScope Global -GroupCategory Security -Path "OU=Security Groups,DC=mednet,DC=lab"
```

Users were assigned to their groups:

```powershell
Add-ADGroupMember -Identity "Clinical-Physicians" -Members "s.mitchell","j.ortega"
Add-ADGroupMember -Identity "Clinical-Nursing" -Members "l.nguyen"
Add-ADGroupMember -Identity "Clinical-Pharmacy" -Members "m.evans"
Add-ADGroupMember -Identity "Admin-HR" -Members "k.booth"
Add-ADGroupMember -Identity "Admin-Finance" -Members "t.reyes"
Add-ADGroupMember -Identity "Admin-Reception" -Members "d.cole"
Add-ADGroupMember -Identity "IT-Staff" -Members "a.turner"
```

> **Note:** The `Security Groups` OU path is directly under the domain root (`DC=mednet,DC=lab`) rather than nested under `OU=MedNet`. This is because the OU was placed at the domain root level during initial AD setup. In a production environment it would typically sit inside the top-level managed OU.

---

## Domain-Joined Computers

The following computer accounts currently exist in the domain:

| Computer Name | Type | Location | Status |
|---|---|---|---|
| `WIN-1UKKKVRD HPB` | Domain Controller | Default `Domain Controllers` OU | Active |
| `MEDNET-FS01` | File Server (Debian 12 / Samba) | Default `Computers` container | Active — joined during MedNet-FileServer lab |

> **Note:** `MEDNET-FS01` is currently in the default `Computers` container. In a future cleanup pass it will be moved to `Workstations/Servers` to align with the naming and OU structure defined above, and to allow server-scoped GPOs to apply correctly when Phase 3 machine policies are configured.

---

## GPO Delegation

Each `Departments` sub-OU serves as a natural GPO link point. Department-scoped policy assignments (USB lockdown for Clinical, software restrictions for Administrative, etc.) are covered in detail in [02-gpo-configuration.md](02-gpo-configuration.md).

The `Workstations` OU split allows machine-level GPOs (screen lock timeout, BitLocker enforcement) to be applied to endpoints when Phase 3 Windows VMs are domain-joined, without affecting server objects.

---

## Related Documents

| Document | Description |
|---|---|
| [02-gpo-configuration.md](02-gpo-configuration.md) | GPO design, settings, and enforcement details |
| [03-pki-and-ldaps.md](03-pki-and-ldaps.md) | Internal CA setup, certificate deployment, LDAPS configuration |
| [04-security-hardening.md](04-security-hardening.md) | Account policies, audit configuration, event forwarding |
