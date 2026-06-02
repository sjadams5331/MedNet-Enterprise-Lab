# Setup & Integration — Active Directory (LDAPS) and Hardening

## Purpose

This document describes the deployment of the MedNet IT service desk (osTicket) and its integration with Microsoft Active Directory over LDAP-over-SSL (LDAPS), along with the security hardening applied to the host and application.

The objectives are to centralize authentication, eliminate local credential storage within the helpdesk platform, and reduce attack surface to a level appropriate for an internal hospital IT service desk backed by centralized identity.

End users and support agents authenticate using Active Directory credentials; osTicket functions strictly as an identity consumer.

## Enterprise Authentication Model

Active Directory serves as the **authoritative identity provider** for this environment. osTicket does not manage passwords, validate credentials locally, or act as an identity store.

Key principles:
- Identity authority resides exclusively in Active Directory
- Authentication traffic is encrypted end-to-end
- osTicket performs directory lookups and authorization mapping only
- Credentials are never stored or validated locally by the application

## Directory Environment

### Active Directory

- **Domain:** `mednet.lab`
- **Domain Controller:** `dc01.mednet.lab`
- **IP Address:** `192.168.56.10`
- **Directory Services:** AD DS, DNS
- **Certificate Authority:** Internal Microsoft CA (`MedNet-RootCA`)

### osTicket Application Server

- **Hostname:** `itsm01.mednet.lab`
- **IP Address:** `192.168.56.120`
- **Operating System:** Debian GNU/Linux 12
- **Application:** osTicket v1.18.x

The osTicket server operates as a Linux application consumer within a Windows-centric identity domain, reflecting common enterprise heterogeneity.

## Why LDAPS Is Mandatory

Plain LDAP transmits credentials in a form that can be intercepted on the network. In a hospital environment, plaintext directory authentication is treated as a critical security failure — credentials that protect access to PHI-adjacent systems must never traverse the network in cleartext, even on internal segments.

LDAPS provides:
- Encryption of credentials and directory queries
- Server identity verification using X.509 certificates
- Protection against credential harvesting and replay
- Alignment with enterprise security baselines

This component **does not permit plaintext LDAP** under any circumstances.

## Certificate Trust and PKI

The Domain Controller presents an LDAPS certificate issued by the internal Microsoft Certificate Authority (`MedNet-RootCA`). To validate that identity, the Debian server is configured to trust the internal CA at the system level:

- The CA certificate is installed into the system trust store
- Trust is applied system-wide, not application-specific
- LDAP clients rely on standard TLS validation

Successful TLS validation confirms certificate chain integrity, correct hostname matching, and an encrypted session.

|  |  |
|---|---|
| ![Internal CA trusted in the Debian system store](../screenshots/ticketing_setup_and_integration_01.png) | ![LDAPS connection parameters in osTicket](../screenshots/ticketing_setup_and_integration_02.png) |
| Internal `MedNet-RootCA` trusted in the Debian system trust store | LDAPS connection parameters configured in osTicket |

## Service Account Design

### Account Details

- **Account:** `svc_osticket@mednet.lab`
- **Purpose:** LDAP bind and directory search
- **Permissions:** Read-only

### Least-Privilege Enforcement

The service account:
- Cannot modify directory objects
- Is not a domain administrator and is excluded from privileged groups
- Is denied interactive logon
- Is restricted to authentication and lookup operations only

**Rationale:** Service accounts are high-value targets. Restricting scope limits the blast radius of a credential compromise.

## osTicket LDAP Configuration

### Connection Parameters

| Parameter | Value |
|---|---|
| LDAP URI | `ldaps://dc01.mednet.lab:636` |
| Bind Identity | `svc_osticket@mednet.lab` |
| Base DN | `DC=mednet,DC=lab` |

The connection is encrypted by default and validated against the system trust store.

### Authentication Behavior

- End users and agents authenticate using Active Directory credentials
- User records are created dynamically on first successful login
- Support agents are mapped to application roles after authentication
- Passwords are never stored within osTicket

Directory attributes consumed include `sAMAccountName`, `mail`, `displayName`, and `memberOf`.

### Authentication Flow

1. The user submits credentials to the osTicket portal
2. osTicket binds to Active Directory over LDAPS
3. Active Directory validates the credentials
4. A session is established with no local password storage; for agents, RBAC then determines permitted actions

Authentication (who you are) and authorization (what you may do) are explicitly separated.

## Security Hardening

### Application Hardening

- Local osTicket staff authentication is disabled — all staff authenticate via Active Directory
- No fallback local administrator accounts exist
- Agent permissions are role-based, with roles mapped to Active Directory security groups
- Permissions are denied unless explicitly required (e.g., Tier 1 cannot close tickets or modify SLAs)

**Rationale:** Centralizing identity ensures every action is subject to audit and policy. Bypassing AD would bypass both.

### Host and Network Hardening

- The server resides on the internal host-only network (`192.168.56.0/24`) with no inbound access from external networks
- Only required ports are exposed: HTTPS (internal), LDAPS, and the database (local only)
- Host firewall rules explicitly restrict inbound traffic
- SSH is limited to administrative users; root login over SSH is disabled
- Package updates are applied regularly

**Rationale:** The service desk must not become a lateral-movement pivot during an intrusion.

|  |  |
|---|---|
| ![AD-backed staff list in osTicket](../screenshots/ticketing_setup_and_integration_03.png) | ![Authentication policy with local auth disabled](../screenshots/ticketing_setup_and_integration_04.png) |
| Staff directory populated from Active Directory over LDAPS | Authentication policy enforcing AD-only sign-in |

## Logging

- osTicket logs ticket activity (assignment, status changes, agent responses) and authentication attempts
- Apache access and error logs are enabled on the host
- LDAP authentication events are logged on the Domain Controller

Together these provide attributable, after-the-fact accountability in support of HIPAA audit-control expectations.

## Common Failure Scenarios

| Symptom | Likely Cause |
|---|---|
| LDAPS connection fails | Internal CA not trusted in the system store |
| Certificate validation error | Hostname mismatch or expired certificate |
| Authentication timeout | Firewall or network filtering |
| Intermittent login success | DNS resolution inconsistency (e.g., dual-adapter records) |
| Bind failure | Incorrect service account credentials |

## Validation Summary

LDAPS integration was confirmed through a successful TLS handshake and certificate validation from the Debian host, verified trust of the internal CA, successful user and agent authentication using AD credentials, and confirmed role enforcement within osTicket.
