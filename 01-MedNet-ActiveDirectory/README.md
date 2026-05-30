# MedNet Active Directory

> **Identity, Access, and Policy Management for mednet.lab**
> Windows Server Active Directory deployment serving as the central identity provider for the MedNet Enterprise Lab. All services in the environment authenticate against this domain controller, and all access to resources is governed by the groups, policies, and certificate infrastructure built here.

---

## Overview

The Active Directory environment for `mednet.lab` is modeled after a mid-size hospital's IT infrastructure. The organizational structure reflects real departments found in a healthcare setting — clinical, administrative, IT, and management — with security policies and access controls appropriate for each. This is not a basic AD install. Every design decision is intentional and documented.

`dc01.mednet.lab` serves as the single domain controller and hosts all core identity services for the lab: DNS, LDAPS, Group Policy, and an internal Certificate Authority. Every other service in the MedNet environment — osTicket, Zabbix, Wazuh, the file server, and all workstations — depends on this server.

---

## Server Details

| Property | Value |
|---|---|
| Hostname | `dc01.mednet.lab` |
| Operating System | Windows Server |
| Role | Primary Domain Controller |
| Domain | `mednet.lab` |
| LDAPS Port | 636 |
| DNS | Authoritative for `mednet.lab` |
| Certificate Authority | MedNet-RootCA |

---

## Domain Design

### Organizational Unit Structure

The OU structure mirrors a realistic hospital organizational chart. Users, computers, and service accounts are placed in OUs that reflect their function and department, allowing GPOs to be applied with surgical precision rather than domain-wide. Custom OUs are created at the domain root, separate from the default AD containers.

```
mednet.lab
├── Admin Accounts
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
├── Security Groups
├── Service Accounts
└── Workstations
    ├── Computers   (WS-CLIN-01, WS-ADMIN-01, WS-IT-01)
    └── Servers     (MEDNET-FS01, MEDNET-NOC, MEDNET-OSTICKET, MEDNET-SIEM01)
```

### Security Groups

Access to resources across the environment is controlled through AD security groups rather than individual user permissions. This reflects enterprise best practice and allows access to be modified by group membership alone.

| Group | Purpose |
|---|---|
| `Clinical-Physicians` | Physician staff — access to the `physicians` file share |
| `Clinical-Nursing` | Nursing staff — access to the `nursing` file share |
| `Clinical-Pharmacy` | Pharmacy staff — access to the `pharmacy` file share |
| `Admin-HR` | HR staff — access to the `hr` file share |
| `Admin-Finance` | Finance staff — access to the `finance` file share |
| `Admin-Reception` | Reception staff — access to the `reception` file share |
| `IT-Staff` | IT staff — elevated access across systems |

All custom security groups are Global / Security groups located in the `Security Groups` OU and follow a `Department-Role` naming convention. Group creation and membership are documented in [01-domain-design.md](docs/01-domain-design.md).

---

## Group Policy

GPOs are applied at the OU level and scoped to their intended targets. Policies are designed around the access control and hardening requirements appropriate for a healthcare environment.

### Deployed GPOs

| GPO Name | Linked To | Purpose |
|---|---|---|
| `Default Domain Policy` | `mednet.lab` (domain root) | Password and account lockout policy for all accounts |
| `Clinical-Workstation-Policy` | `Departments/Clinical` | User-side: 5-minute screen lock, Control Panel restriction |
| `Administrative-User-Policy` | `Departments/Administrative` | User-side: 10-minute screen lock, command-prompt restriction |
| `IT-Department-Policy` | `Workstations/Computers` | Computer-side: endpoint audit policy (logon, account management, policy change) |
| `Workstation-Baseline-Policy` | `Workstations/Computers` | Computer-side: removable-storage block, Autoplay off, Windows Firewall, event-log sizing |

User-configuration settings are linked to the department OUs that hold the user accounts; Computer-configuration settings are linked to `Workstations/Computers`, which holds the machine objects. This ensures each policy applies to the correct object type. The domain controller's own audit events are captured by the built-in `Default Domain Controllers Policy`. Full settings and rationale are in [02-gpo-configuration.md](docs/02-gpo-configuration.md).

### Password Policy

| Setting | Value |
|---|---|
| Minimum password length | 12 characters |
| Complexity requirement | Enabled |
| Maximum password age | 90 days |
| Account lockout threshold | 5 attempts |
| Lockout duration | 30 minutes |
| Lockout observation window | 30 minutes |

---

## Internal PKI / Certificate Authority

An internal Certificate Authority was deployed under the domain to enable LDAPS and support certificate-based trust across services. All services that communicate with Active Directory over LDAPS trust the MedNet-RootCA.

| Property | Value |
|---|---|
| CA Name | MedNet-RootCA |
| CA Type | Enterprise Root CA |
| Certificate issued to | `dc01.mednet.lab` |
| LDAPS Port | 636 |

The CA certificate is distributed to Linux-based service VMs via manual trust store configuration, enabling encrypted and validated LDAP communication from osTicket, Zabbix, and Wazuh.

---

## LDAPS Integration

All services that authenticate users against Active Directory do so over LDAPS on port 636. Plaintext LDAP on port 389 is not used. Each service uses a dedicated read-only service account scoped to directory lookups only.

### Service Accounts

| Account | Used By | Permissions |
|---|---|---|
| `svc_osticket@mednet.lab` | osTicket | Read-only directory access |
| `svc_zabbix@mednet.lab` | Zabbix | Read-only directory access |
| `svc_wazuh@mednet.lab` | Wazuh | Read-only directory access |

All service accounts are placed in the `Service Accounts` OU, have passwords set to non-expiring, and are restricted from interactive login via GPO.

---

## Security Hardening

### Account Security
- Password policy enforced at domain level
- Account lockout policy active on all accounts
- Administrator account renamed and restricted
- Guest account disabled
- Service accounts restricted to their designated function only

### Audit Policy
The following events are audited and logged to the Windows Security event log. Logs are forwarded to Wazuh for correlation and alerting.

- Account logon and logoff events
- Account management changes
- Privilege use
- Object access
- Policy changes
- System events

### Event Forwarding
Windows event logs from `dc01.mednet.lab` are ingested by the Wazuh SIEM for correlation, alerting, and long-term retention. This mirrors a production SOC workflow where the domain controller is a high-priority log source.

---

## Integration Points

The domain controller is the dependency anchor for the entire MedNet environment. Every service integrates with it in some form.

| Service | Integration |
|---|---|
| osTicket (`mednet-osticket.mednet.lab`) | LDAPS user authentication, agent accounts sourced from AD |
| Zabbix (`mednet-noc.mednet.lab`) | LDAPS authentication for monitoring console access |
| Wazuh (`mednet-siem01.mednet.lab`) | Event log ingestion, agent deployment to domain endpoints |
| File Server (`mednet-fs01.mednet.lab`) | AD-integrated Samba shares, group-based permissions |
| Workstations (`ws-clin-01`, `ws-admin-01`, `ws-it-01`) | Domain-joined, GPO-managed, Wazuh agent deployed |

---

## Validation

The following were verified to confirm the environment is functioning as designed:

- Domain join successful for all servers and workstations
- LDAPS connectivity confirmed from all Linux-based service VMs on port 636
- GPOs applying correctly — verified via `gpresult /r` on workstations
- Service accounts authenticating successfully from osTicket, Zabbix, and Wazuh
- Audit events generating in Security event log and appearing in Wazuh

---

## Documentation

| Document | Description |
|---|---|
| [01-domain-design.md](docs/01-domain-design.md) | OU structure, naming conventions, hospital org model |
| [02-gpo-configuration.md](docs/02-gpo-configuration.md) | GPO design, settings, and enforcement details |
| [03-pki-and-ldaps.md](docs/03-pki-and-ldaps.md) | Internal CA setup, certificate deployment, LDAPS configuration |
| [04-security-hardening.md](docs/04-security-hardening.md) | Account policies, audit configuration, event forwarding |

---

*Part of the [MedNet Enterprise Lab](../README.md) — Enterprise Healthcare IT Infrastructure & Security Operations Home Lab*
