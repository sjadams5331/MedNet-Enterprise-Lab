# 02 — GPO Configuration
 
## Overview
 
This document covers the Group Policy Object (GPO) design and configuration for the MedNet Enterprise Lab. Five GPOs are implemented across the domain to enforce security baselines, simulate healthcare compliance requirements (HIPAA), and demonstrate layered policy design across different OU scopes.
 
---
 
## GPO Summary
 
| GPO Name | Linked To | Scope |
|---|---|---|
| Default Domain Policy | `mednet.lab` (domain root) | All users and computers |
| Clinical-Workstation-Policy | `Departments/Clinical` | Clinical staff users |
| Administrative-User-Policy | `Departments/Administrative` | Administrative staff users |
| IT-Department-Policy | `Departments/IT` | IT staff computers |
| Workstation-Baseline-Policy | `Workstations/Computers` | Domain-joined endpoints |
 
---
 
## GPO #1 — Default Domain Policy (Password & Lockout)
 
The Default Domain Policy is configured at the domain root and applies to all user accounts in `mednet.lab`. It enforces the baseline password and account lockout standards required across the organization.
 
### Password Policy Settings
 
| Setting | Value |
|---|---|
| Enforce password history | 10 passwords remembered |
| Maximum password age | 90 days |
| Minimum password age | 1 day |
| Minimum password length | 12 characters |
| Password must meet complexity requirements | Enabled |
| Store passwords using reversible encryption | Disabled |
 
### Account Lockout Settings
 
| Setting | Value |
|---|---|
| Account lockout threshold | 5 invalid logon attempts |
| Account lockout duration | 30 minutes |
| Reset account lockout counter after | 30 minutes |
| Allow Administrator account lockout | Enabled |
 
| | |
|---|---|
| ![Password Policy](screenshots/02-gpo-configuration.md_01.png) | ![Lockout Policy](screenshots/02-gpo-configuration.md_02.png) |
 
---
 
## GPO #2 — Clinical-Workstation-Policy
 
Linked to `Departments/Clinical`. This GPO enforces strict workstation controls for clinical staff in alignment with HIPAA workstation security requirements. Clinical environments handle protected health information (PHI) and require tighter access controls than other departments.
 
### Settings
 
| Setting | Value | Configuration Side |
|---|---|---|
| Enable screen saver | Enabled | User |
| Screen saver timeout | 300 seconds (5 minutes) | User |
| Password protect the screen saver | Enabled | User |
| All Removable Storage classes: Deny all access | Enabled | Computer |
| Prohibit access to Control Panel and PC Settings | Enabled | User |
 
### Design Rationale
 
The 5-minute screen lock timeout reflects HIPAA's workstation use standards, which require that systems left unattended in clinical areas are secured. USB storage is disabled entirely to prevent unauthorized data exfiltration of patient records. Control Panel access is restricted to prevent clinical users from modifying system or network settings.
 
| | |
|---|---|
| ![Clinical Screen Lock](screenshots/02-gpo-configuration.md_03.png) | ![Clinical USB Block](screenshots/02-gpo-configuration.md_04.png) |
 
![Clinical Control Panel](screenshots/02-gpo-configuration.md_05.png)
 
---
 
## GPO #3 — Administrative-User-Policy
 
Linked to `Departments/Administrative`. This GPO applies moderate restrictions to administrative staff (HR, Finance, Reception). These users handle sensitive organizational data but require more flexibility than clinical staff.
 
### Settings
 
| Setting | Value | Configuration Side |
|---|---|---|
| Enable screen saver | Enabled | User |
| Screen saver timeout | 600 seconds (10 minutes) | User |
| Password protect the screen saver | Enabled | User |
| Prevent access to the command prompt | Enabled | User |
 
### Design Rationale
 
The 10-minute screen lock balances security with usability for desk-based administrative work. CMD and PowerShell access is restricted to reduce the attack surface for non-technical users — administrative staff have no legitimate need for command-line tools, and restricting them limits the effectiveness of phishing-delivered payloads.
 
| | |
|---|---|
| ![Admin Screen Lock](screenshots/02-gpo-configuration.md_06.png) | ![Admin CMD Restriction](screenshots/02-gpo-configuration.md_07.png) |
 
---
 
## GPO #4 — IT-Department-Policy
 
Linked to `Departments/IT`. IT staff are intentionally unrestricted on the user side — helpdesk and systems engineers require full access to tools and settings to perform their roles. The focus of this GPO is audit logging to ensure IT actions are tracked and accountable.
 
### Audit Policy Settings
 
| Setting | Value |
|---|---|
| Audit account logon events | Success, Failure |
| Audit logon events | Success, Failure |
| Audit privilege use | Success, Failure |
| Audit policy change | Success, Failure |
| Audit account management | Success, Failure |
 
### Design Rationale
 
Auditing IT accounts is a core principle of privileged access management. Even trusted administrators should have their actions logged — this supports incident response, change tracking, and compliance reporting. These events feed into the Wazuh SIEM deployed in Phase 2.
 
| | |
|---|---|
| ![IT Audit Policy 1](screenshots/02-gpo-configuration.md_08.png) | ![IT Audit Policy 2](screenshots/02-gpo-configuration.md_09.png) |
 
---
 
## GPO #5 — Workstation-Baseline-Policy
 
Linked to `Workstations/Computers`. This GPO establishes the security baseline for all domain-joined endpoints and will apply automatically to Phase 3 Windows VMs when they are joined to the domain. It is configured now so that policy enforcement is in place before any endpoints are added.
 
### Settings
 
| Setting | Value |
|---|---|
| Turn off Autoplay | Enabled — All Drives |
| Disallow Autoplay for non-volume devices | Enabled |
| Windows Firewall — Domain Profile | On |
| Windows Firewall — Private Profile | On |
| Windows Firewall — Public Profile | On |
| Maximum application log size | 32768 KB |
| Maximum security log size | 81920 KB |
| Maximum system log size | 32768 KB |
 
### Design Rationale
 
Disabling Autorun/Autoplay eliminates a common malware delivery vector (e.g., malicious USB drives). Enforcing Windows Firewall across all profiles ensures endpoints are protected regardless of network context. Expanded event log sizes prevent log rollover on active systems, ensuring security events are retained long enough for investigation — the security log is sized larger to accommodate the higher volume of audit events.
 
| | |
|---|---|
| ![Workstation Autoplay](screenshots/02-gpo-configuration.md_10.png) | ![Workstation Firewall](screenshots/02-gpo-configuration.md_11.png) |
 
---
 
## Related Documents
 
| Document | Description |
|---|---|
| [01-domain-design.md](01-domain-design.md) | OU structure, naming conventions, hospital org model |
| [03-pki-and-ldaps.md](03-pki-and-ldaps.md) | Internal CA setup, certificate deployment, LDAPS configuration |
| [04-security-hardening.md](04-security-hardening.md) | Account policies, audit configuration, event forwarding |
