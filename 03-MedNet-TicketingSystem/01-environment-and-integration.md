 Environment & Integration
 
## Overview
 
The MedNet osTicket deployment runs on a dedicated Debian GNU/Linux 12 (Bookworm) virtual machine and serves as the IT service management platform for the `mednet.lab` environment. It is the primary interface for end users to submit support requests and for help desk staff to manage, route, and resolve tickets.
 
Rather than operating as a standalone tool, osTicket is fully integrated into the MedNet ecosystem. Users authenticate using their Active Directory credentials, agents are sourced from AD security groups, and monitoring alerts from Zabbix are automatically converted into tickets — creating a closed-loop workflow between infrastructure monitoring and service desk operations.
 
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
| LDAPS Integration | `dc01.mednet.lab` on port 636 |
 
---
 
## Active Directory Integration
 
osTicket authenticates both end users and agents against Active Directory over LDAPS. No local osTicket accounts are used for staff — all agent logins are sourced from AD, ensuring a single identity across the MedNet environment.
 
### LDAPS Configuration
 
| Property | Value |
|---|---|
| Directory Server | `dc01.mednet.lab` |
| Port | 636 (LDAPS) |
| Service Account | `svc-osticket@mednet.lab` |
| Service Account Permissions | Read-only directory access |
| CA Trust | MedNet Internal CA installed on `itsm01.mednet.lab` |
 
The Debian server trusts the MedNet Internal CA, enabling encrypted and validated communication with the domain controller. Plaintext LDAP on port 389 is not used.
 
### Authentication Model
 
- End users submit tickets via the web portal using Active Directory credentials
- Help desk agents log into the staff panel using Active Directory accounts
- The service account is scoped to directory reads only and cannot log in interactively
---
 
> 📸 **Screenshot:** osTicket LDAP settings page showing `dc01.mednet.lab`, port 636, and service account configured
 
> 📸 **Screenshot:** Successful AD-authenticated agent login to the staff panel
 
---
 
## Network Integration
 
`itsm01.mednet.lab` is domain-aware and communicates with the following MedNet services:
 
| Service | Purpose |
|---|---|
| `dc01.mednet.lab` | LDAPS authentication and identity |
| `mon01.mednet.lab` | Receives automated tickets from Zabbix threshold alerts |
 
---
 
## Related Documentation
 
- [02-helpdesk-configuration.md](02-helpdesk-configuration.md) — Departments, roles, help topics, and SLA configuration
- [03-zabbix-integration.md](03-zabbix-integration.md) — Automated ticket creation from Zabbix monitoring alerts
- [Original osTicket Lab](https://github.com/sjadams5331/osTicket-Helpdesk-Home-Lab) — Full standalone build and configuration documentation
---
 
*Part of the [MedNet Enterprise Lab](../../README.md)*
