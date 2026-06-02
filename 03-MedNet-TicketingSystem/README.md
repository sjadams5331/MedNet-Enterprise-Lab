# MedNet IT Service Desk

## Overview

This component provides the centralized IT service desk for the MedNet Health Systems environment. It is the single intake point for IT incidents, service requests, and access requests across the hospital's clinical, administrative, and IT user populations.

The service desk is intentionally positioned as the **operational convergence point** of the lab: monitoring and security telemetry surface here as actionable, auditable tickets, and all support activity is recorded against a tiered workflow with enforced separation of duties.

## Platform Details

| Attribute | Value |
|---|---|
| Hostname | `itsm01.mednet.lab` |
| IP Address | `192.168.56.120` |
| Operating System | Debian GNU/Linux 12 |
| Application | osTicket v1.18.x |
| Authentication | Active Directory via LDAPS (`ldaps://dc01.mednet.lab:636`) |
| Identity Provider | `dc01.mednet.lab` (`192.168.56.10`) |

osTicket runs as a Linux application consumer inside a Windows-centric identity domain (`mednet.lab`), reflecting the platform heterogeneity common to real hospital IT environments.

## Role in the Lab

The service desk does not operate in isolation. It is where the lab's monitoring and security tiers become operational work:

- **Zabbix** (`mon01.mednet.lab`) forwards performance and availability signals — service outages, resource exhaustion, and agent check-in failures.
- **Wazuh** (`siem01.mednet.lab`) forwards security events — authentication anomalies, file-integrity changes, and policy violations.

Both feed into the service desk independently; they do not communicate with each other directly. osTicket is the human-facing layer that turns machine telemetry into tracked, assignable, and auditable work.

```
   Zabbix (performance/availability) ─┐
                                      ├──► osTicket (itsm01) ──► tiered triage / resolution
   Wazuh  (security events) ──────────┘
```

## HIPAA Alignment

In a hospital environment, the service desk is a compliance artifact as much as an operational tool. This deployment supports HIPAA expectations through:

- **Audit Controls (§164.312(b))** — every assignment, status change, and resolution is recorded against an authenticated Active Directory identity.
- **Information Access Management (§164.308(a)(4))** — access and group-membership changes are requested, authorized, and fulfilled through a documented ticket trail rather than ad hoc, supporting the minimum-necessary principle for systems that touch PHI.
- **Accountability** — no anonymous or local fallback accounts exist; every agent action is attributable to a directory identity.

## Documentation Index

| Document | Contents |
|---|---|
| `ticketing_setup_and_integration.md` | Deployment, AD/LDAPS authentication, service account design, and security hardening |
| `ticketing_workflows.md` | Ticket lifecycle, tiered authority, escalation, and closure standards |
