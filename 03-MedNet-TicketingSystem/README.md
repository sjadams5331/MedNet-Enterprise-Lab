# MedNet osTicket
 
> **IT Service Management Platform for mednet.lab**
> Enterprise osTicket deployment serving as the central helpdesk and service management platform for the MedNet healthcare environment. Fully integrated with Active Directory for authentication, organized around a realistic hospital service desk structure, and connected to Zabbix for automated infrastructure alert ticketing.
 
---
 
## Overview
 
`itsm01.mednet.lab` runs osTicket on Debian GNU/Linux 12 (Bookworm) and handles all IT service requests across the MedNet environment. The deployment goes beyond a basic install — it reflects how a real hospital service desk operates, with department-based ticket routing, SLA tiers calibrated to healthcare urgency, and role-based access control sourced from Active Directory.
 
The most operationally significant aspect of this deployment is its integration with Zabbix. When infrastructure monitoring detects a threshold breach on any MedNet host, a ticket is automatically created in osTicket and routed to the NOC / Infrastructure department. This closes the loop between monitoring and service management without any manual intervention.
 
This service builds directly on the standalone osTicket lab documented separately. Within MedNet the focus shifts from build and configuration to integration and operational workflow.
 
---
 
## Server Details
 
| Property | Value |
|---|---|
| Hostname | `itsm01.mednet.lab` |
| Operating System | Debian GNU/Linux 12 (Bookworm) |
| Web Server | Apache 2 |
| Application Stack | PHP 8.2 |
| Database | MariaDB |
| Ticketing Platform | osTicket v1.18.x |
| LDAPS Integration | `dc01.mednet.lab` port 636 |
 
---
 
## Architecture
 
osTicket sits at the intersection of identity, monitoring, and service operations in the MedNet environment. It depends on Active Directory for authentication and receives automated input from Zabbix, making it a central coordination point for IT operations.
 
```
         ┌─────────────────────┐
         │   dc01.mednet.lab   │
         │   Active Directory  │
         │   LDAPS / Port 636  │
         └──────────┬──────────┘
                    │ Authentication
                    ▼
         ┌─────────────────────┐
         │   itsm01.mednet.lab │◄──── Automated alerts ──── mon01.mednet.lab
         │   osTicket ITSM     │                             (Zabbix)
         │   Apache / MariaDB  │
         └─────────────────────┘
                    │
         ┌──────────┴──────────┐
         │                     │
    End Users              Help Desk Agents
  (AD credentials)        (AD credentials)
```
 
---
 
## Key Configuration
 
### Departments
 
| Department | Responsibility |
|---|---|
| Service Desk | General end user support, password resets, hardware issues |
| NOC / Infrastructure | Network and server tickets — receives automated Zabbix alerts |
| Clinical Systems | Clinical application and device issues |
| Security | Security incidents, access requests, policy violations |
 
### SLA Tiers
 
| Tier | Priority | Grace Period | Schedule |
|---|---|---|---|
| P1 — Critical | Critical | 1 hour | 24/7 |
| P2 — High | High | 4 hours | 24/7 |
| P3 — Normal | Normal | 8 hours | Business hours |
| P4 — Low | Low | 24 hours | Business hours |
 
### Roles
 
| Role | Scope |
|---|---|
| Tier 1 Support | Standard ticket handling within assigned department |
| Tier 2 Support | Escalation, reassignment, SLA overrides |
| NOC / Infrastructure | Infrastructure queues, limited user data access |
| Administrator | Full system configuration |
 
---
 
## Integration Points
 
| Service | Integration |
|---|---|
| `dc01.mednet.lab` | LDAPS authentication for all users and agents |
| `mon01.mednet.lab` | Zabbix sends automated tickets via osTicket REST API on threshold alerts |
 
---
 
## Validation
 
- AD-authenticated login confirmed for both end users and agents
- Department routing verified across all help topics
- SLA enforcement confirmed — overdue tickets flagged correctly in agent view
- Automated ticket creation from Zabbix confirmed end-to-end
- Role-based permissions verified — Tier 1 cannot reassign or override SLAs
---
 
> 📸 **Screenshot:** osTicket staff panel showing active ticket queue with department routing and SLA status
 
> 📸 **Screenshot:** End user portal showing help topic selection and AD-authenticated login
 
---
 
## Documentation
 
| Document | Description |
|---|---|
| [01-environment-and-integration.md](docs/01-environment-and-integration.md) | Server details, LDAPS configuration, AD integration |
| [02-helpdesk-configuration.md](docs/02-helpdesk-configuration.md) | Departments, roles, help topics, SLA tiers, ticket lifecycle |
| [03-zabbix-integration.md](docs/03-zabbix-integration.md) | Automated ticket creation from Zabbix monitoring alerts |
 
For full build and installation documentation see the [original osTicket standalone lab](https://github.com/sjadams5331/osTicket-Helpdesk-Home-Lab).
 
---
 
## Skills Demonstrated
 
- Linux server administration (Debian 12)
- osTicket deployment and production configuration
- Active Directory integration via LDAPS
- Healthcare-aligned service desk design
- SLA design and enforcement
- Role-based access control for ITSM
- REST API integration between monitoring and ticketing platforms
- Closed-loop NOC alert-to-ticket workflow
---
 
*Part of the [MedNet Enterprise Lab](../README.md) — Enterprise Healthcare IT Infrastructure & Security Operations Home Lab*
