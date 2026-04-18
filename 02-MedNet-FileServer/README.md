# MedNet-FileServer
 
## Overview
 
This lab documents the deployment and configuration of a domain-joined Samba file server simulating a healthcare enterprise file storage environment. The server is a member of the `mednet.lab` Active Directory domain and provides department-scoped SMB shares mapped to AD security groups, reflecting a realistic hospital file access model.
 
This VM is part of the broader **MedNet-Enterprise-Lab** — a simulated hospital IT infrastructure built for portfolio and skills development purposes.
 
---
 
## Environment Details
 
| Setting | Value |
|---|---|
| Hostname | `mednet-fs01` |
| FQDN | `mednet-fs01.mednet.lab` |
| IP Address | `192.168.56.20` |
| Operating System | Debian 12 (Bookworm) — Headless Server |
| Domain | `mednet.lab` |
| Role | Samba File Server — Domain Member |
| Samba Version | 4.17.12-Debian |
 
---
 
## Prerequisites
 
The following must be running and accessible before this lab is built:
 
| Dependency | Details |
|---|---|
| AD Domain Controller | `WIN-1UKKKVRD HPB.mednet.lab` at `192.168.56.10` |
| Domain | `mednet.lab` with functional Kerberos |
| Internal CA | Available on the AD server for certificate issuance |
| AD Security Groups | Department groups created in `OU=Security Groups,DC=mednet,DC=lab` |
| VirtualBox Host-Only Network | Shared with all other lab VMs on `192.168.56.0/24` |
 
---
 
## Lab Structure
 
```
MedNet-FileServer/
├── README.md                   ← This file
├── 01-samba-installation.md    ← Install, verify, initial config
├── 02-domain-join.md           ← Domain join to mednet.lab
├── 03-share-configuration.md   ← Hospital share structure, smb.conf, AD permissions
├── 04-security-hardening.md    ← SMB signing, firewall, SSH hardening
└── screenshots/
```
 
---
 
## Hospital Share Structure
 
| Share | Department | AD Group |
|---|---|---|
| `physicians` | Clinical / Physicians | `Clinical-Physicians` |
| `nursing` | Clinical / Nursing | `Clinical-Nursing` |
| `pharmacy` | Clinical / Pharmacy | `Clinical-Pharmacy` |
| `hr` | Administrative / HR | `Admin-HR` |
| `finance` | Administrative / Finance | `Admin-Finance` |
| `reception` | Administrative / Reception | `Admin-Reception` |
| `it` | IT Department | `IT-Staff` |
| `shared` | All Staff | `Domain Users` |
 
Access to each share is restricted to the corresponding AD security group, enforcing role-based access control aligned with HIPAA minimum necessary access principles.
 
---
 
## Key Technologies
 
- **Samba 4** — SMB/CIFS file sharing on Linux
- **Kerberos** — Authentication via `mednet.lab` AD domain
- **realmd / adcli** — Domain join toolchain
- **SSSD** — System Security Services Daemon for AD integration
- **UFW** — Host-based firewall
- **SMB2+** — Minimum protocol enforced, SMB1 disabled
- **SMB Signing** — Mandatory on both server and client
---
 
## Related Lab Sections
 
| Lab | Description |
|---|---|
| [MedNet-ActiveDirectory](../MedNet-ActiveDirectory/) | Domain controller, GPOs, PKI, OU structure |
| [MedNet-Zabbix](../MedNet-Zabbix/) | Network monitoring — file server added as monitored host |
| [MedNet-Wazuh](../MedNet-Wazuh/) | SIEM — Wazuh agent to be deployed on this VM in Phase 2 |
| [MedNet-Workstations](../MedNet-Workstations/) | Windows endpoints that will map drives to these shares |

---
 
*Part of the [MedNet Enterprise Lab](../README.md) — Enterprise Healthcare IT Infrastructure & Security Operations Home Lab*
