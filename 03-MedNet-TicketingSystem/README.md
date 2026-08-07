# MedNet Ticketing System

> **AD-Integrated Helpdesk & ITSM Platform for mednet.lab**
> An osTicket deployment on Debian, authenticating staff against Active Directory over LDAPS while keeping the client-facing support portal locally managed. Models the IT service desk tier of a hospital environment, with department-based routing, differentiated SLAs by clinical impact, and TLS secured end to end using the lab's existing internal CA.

---

## Overview

`itsm01` runs osTicket as the MedNet Enterprise Lab's ITSM platform. Unlike the file server, it is **not domain-joined**: Active Directory integration happens entirely at the application layer, with staff authenticating through osTicket's built-in LDAP plugin over LDAPS rather than through host-level domain membership. This keeps the deployment's scope deliberately narrow: one dependency on AD (staff login), no dependency on a mail server, and no unnecessary attack surface from joining a domain the application doesn't otherwise need.

The service desk structure reflects the same hospital-context design used throughout the lab. A dedicated Clinical Systems department carries a materially tighter, 24/7 service level agreement than general IT requests, the ticketing system's version of the access-compartmentalization principle used in the file server and AD modules.

This module also documents its own troubleshooting honestly, including several real infrastructure gaps found and fixed during TLS setup (a missing DNS record, a non-persistent network interface, and a directory permission gap) rather than presenting a cleaned-up version of events.

---

## Environment Details

| Setting | Value |
|---|---|
| Hostname | `itsm01` |
| FQDN | `itsm01.mednet.lab` |
| IP Address | `192.168.56.120` |
| Operating System | Debian 12 (Bookworm), Headless Server |
| Domain | `mednet.lab` (LDAPS integration only, **not** domain-joined) |
| Role | osTicket Helpdesk / ITSM Platform |
| Application Stack | Apache 2, PHP 8.2, MariaDB |
| osTicket Version | v1.18.1 |
| TLS | Certificate issued by `MedNet-RootCA`, Force HTTPS enforced |

---

## Prerequisites

| Dependency | Details |
|---|---|
| AD Domain Controller | `dc01.mednet.lab` at `192.168.56.10` |
| Domain | `mednet.lab`, LDAPS reachable on port 636 |
| Internal CA | `MedNet-RootCA`, used for both LDAPS trust and the TLS certificate issued to `itsm01` |
| LDAP Bind Account | `svc_osticket@mednet.lab` (dedicated, least-privilege, search/bind only) |
| VirtualBox Host-Only Network | Shared with all lab VMs on `192.168.56.0/24` |

---

## Architecture

```
   AD Domain Controller                    ITSM Platform (LDAPS client only)
   dc01.mednet.lab                          itsm01.mednet.lab
   192.168.56.10                            192.168.56.120
        │                                          │
        │   LDAPS (636), svc_osticket bind         │
        └──────────────────────────────────────────┘
                             │
        Staff authenticate via AD; the public portal is locally managed
                             │
        ┌────────────────────┼────────────────────┐
   Help Desk           Network & Infra       Clinical Systems
   (Public, default)   (Public)              (Public, 4hr/24-7 SLA)
```

Staff authentication is delegated to Active Directory; the client-facing support portal is not, since patients and end users submitting tickets have no need for a domain identity. A separate, local break-glass administrative account is isolated in a Private `Admins` department, used only if AD/LDAPS is ever unavailable.

---

## Service Desk Structure

| Department | Type | Agent | SLA |
|---|---|---|---|
| Help Desk (default) | Public | Samuel Adams (`s.adams`) | Standard Support SLA (24hr, business hours) |
| Network & Infrastructure | Public | Alex Turner (`a.turner`) | Standard Support SLA (24hr, business hours) |
| Clinical Systems | Public | Mike Reyes | Clinical Systems SLA (4hr, 24/7) |
| Admins | Private | Local break-glass account | N/A |

Five Help Topics route client-submitted tickets to the correct department automatically, and cross-department escalation (tested end to end, including an SLA correction caught by that testing) is documented in `03-ticket-workflows.md`.

---

## Repository Structure

```
03-MedNet-TicketingSystem/
├── README.md                              ← You are here
├── 01-deployment-and-authentication.md    ← Debian/Apache/MariaDB deployment, AD/LDAPS integration
├── 02-service-desk-structure.md           ← Departments, agents, Help Topics, SLA plans
├── 03-ticket-workflows.md                 ← Lifecycle, escalation, planned monitoring/SIEM integration
├── 04-security-and-backup.md              ← TLS, agent authentication hardening, backup/restore
└── screenshots/
```

---

## Skills Demonstrated

- Linux server administration and LAMP stack deployment on headless Debian
- Application-layer LDAPS authentication integration without domain-joining the host
- Internal PKI certificate issuance and Apache TLS configuration off an existing enterprise CA
- Service desk design: department structure, Help Topic routing, and SLA differentiation by business impact
- Ticket lifecycle management and cross-department escalation, validated with real test tickets
- Root-cause troubleshooting across DNS, NetworkManager interface persistence, Samba/Kerberos authentication chains, and Linux directory permission models
- Database backup automation with a tested, verified restore procedure and least-privilege DB user scoping
- Design documentation for automated ticket creation from monitoring/SIEM alerts, honestly scoped as planned rather than built

---

## Documentation

| Document | Description |
|---|---|
| [01-deployment-and-authentication.md](01-deployment-and-authentication.md) | Debian/Apache/MariaDB deployment and AD/LDAPS authentication integration |
| [02-service-desk-structure.md](02-service-desk-structure.md) | Departments, AD-authenticated agents, Help Topics, and SLA plans |
| [03-ticket-workflows.md](03-ticket-workflows.md) | Ticket lifecycle, cross-department escalation, and planned monitoring/SIEM integration |
| [04-security-and-backup.md](04-security-and-backup.md) | TLS encryption, agent authentication hardening, and tested backup/restore |

---

*Part of the [MedNet Enterprise Lab](../README.md), an Enterprise Healthcare IT Infrastructure & Security Operations home lab.*
