# MedNet IT Service Desk

> **AD-Integrated osTicket Service Desk for mednet.lab**
> A Debian-hosted osTicket deployment that authenticates every user and agent against Active Directory over LDAPS, and serves as the operational convergence point of the lab — where monitoring and security telemetry become tracked, auditable work. Models the IT service management tier of a hospital environment, with role separation and a full per-ticket audit trail aligned to HIPAA accountability and access-management expectations.

---

## Overview

`itsm01` is a headless Debian 12 server running osTicket on an Apache / PHP / MariaDB stack. It is the single intake point for IT incidents, service requests, and access requests across the hospital's clinical, administrative, and IT user populations. Rather than maintaining a local account database, it authenticates users and agents against Active Directory over LDAPS — the same identities used everywhere else in the lab — so every action is attributable to a directory identity and no anonymous or local fallback agent accounts exist.

The service desk is intentionally positioned as the **operational convergence point** of the lab: performance signals from Zabbix and security events from Wazuh both surface here as actionable, auditable tickets, and all support activity is recorded against a tiered workflow with enforced separation of duties. It illustrates a Linux application operating inside a Windows-centric identity domain — the platform heterogeneity common to real hospital IT environments.

This VM is part of the broader **MedNet Enterprise Lab** — a simulated hospital IT infrastructure built for portfolio and skills-development purposes.

---

## Environment Details

| Setting | Value |
|---|---|
| Hostname | `itsm01` |
| FQDN | `itsm01.mednet.lab` |
| IP Address | `192.168.56.120` |
| Operating System | Debian 12 (Bookworm) — Headless Server |
| Domain | `mednet.lab` |
| Role | IT Service Desk (osTicket) — AD-Integrated Application |
| Application | osTicket v1.18.x |
| Web / Data Stack | Apache 2 · PHP 8.2 · MariaDB |
| Authentication | Active Directory via LDAPS (`ldaps://dc01.mednet.lab:636`) |

---

## Prerequisites

The following lab components must be running and reachable before this server is built or used:

| Dependency | Details |
|---|---|
| AD Domain Controller | `dc01.mednet.lab` at `192.168.56.10`, with LDAPS published on `636` |
| Internal CA | `MedNet-RootCA` — its certificate trusted on the Debian host for LDAPS validation |
| Service Account | `svc_osticket@mednet.lab` — least-privilege, directory read-only bind |
| AD Security Groups | Department/role groups in `OU=Security Groups,DC=mednet,DC=lab` mapped to agent roles |
| Monitoring & SIEM | `mon01.mednet.lab` (Zabbix) and `siem01.mednet.lab` (Wazuh) reachable for alert ingestion |
| VirtualBox Host-Only Network | Shared with all lab VMs on `192.168.56.0/24` |

---

## Architecture

The service desk authenticates against Active Directory over an encrypted LDAPS channel rather than holding its own credentials, and it does not operate in isolation. It is where the lab's monitoring and security tiers become operational work: Zabbix forwards performance and availability signals (service outages, resource exhaustion, agent check-in failures) and Wazuh forwards security events (authentication anomalies, file-integrity changes, policy violations). Both are designed to feed in **independently** — they do not communicate with each other — with osTicket as the human-facing layer that turns machine telemetry into tracked, assignable, and auditable work. Alert ingestion from Zabbix and Wazuh is the next integration phase for this module (documented in `05-monitoring-and-siem-integration.md`); the diagram below reflects the intended design.

```
                      AD Domain Controller
                      dc01.mednet.lab — 192.168.56.10
                               │
                               │  LDAPS :636  (user + agent authentication)
                               ▼
   Zabbix (mon01) ─────┐   ┌──────────────────────────────┐
   perf / availability │   │   osTicket  —  itsm01         │
                       ├──►│   192.168.56.120 (Debian 12)  │──►  tiered triage → resolution
   Wazuh  (siem01) ────┘   │   AD-authenticated agents     │      (per-ticket audit trail)
   security events         └──────────────────────────────┘
   (independent feeds, converging here)
```

---

## Service Desk Structure

Agent authority is modeled around tiered roles that enforce separation of duties. Every agent signs in with their Active Directory identity, and queue/department scoping enforces the minimum-necessary boundary — an agent sees only the work appropriate to their role.

| Role | Operational Scope | Access Boundary |
|---|---|---|
| Tier 1 Service Desk | First-line intake, triage, basic resolution | User-facing department queues only |
| Tier 2 / Escalations | Complex incidents, reassignment, priority/SLA changes | Cross-queue within assigned departments |
| Infrastructure / NOC | Monitoring- and security-generated tickets, infrastructure work | Infra queues; minimal exposure to PHI-adjacent data |
| Administrator | Platform configuration and management | Full system administration |

---

## HIPAA Alignment

In a hospital environment the service desk is a compliance artifact as much as an operational tool. This deployment supports HIPAA expectations through:

- **Audit Controls (§164.312(b))** — every assignment, status change, and resolution is recorded against an authenticated AD identity, producing a per-ticket audit trail.
- **Information Access Management (§164.308(a)(4))** — access and group-membership changes are requested, authorized, and fulfilled through a documented ticket trail rather than ad hoc, supporting the minimum-necessary principle for systems that touch PHI.
- **Accountability** — no anonymous or local fallback accounts; every agent action is attributable to a directory identity over an encrypted channel.

---

## Repository Structure

```
03-MedNet-TicketingSystem/
├── README.md                                 ← You are here
├── docs/
   ├── 01-deployment.md                       ← Debian host, LAMP stack, osTicket install
   ├── 02-ad-authentication.md                ← LDAPS auth, svc_osticket, CA trust, user/agent binding
   ├── 03-service-desk-structure.md           ← Departments, teams, roles, help topics, routing
   ├── 04-ticket-workflows.md                 ← Ticket lifecycle, SLAs, escalation, canned responses
   ├── 05-monitoring-and-siem-integration.md  ← Zabbix + Wazuh alert ingestion (convergence point) — planned
   ├── 06-security-hardening.md               ← TLS, install-dir removal, permissions, DB scoping, SSH/host hardening
   └── 07-backup-and-recovery.md              ← Backup method, retention, tested restore
```

---

## Skills Demonstrated

- Deploying and operating a Linux web application stack (Apache / PHP / MariaDB) on headless Debian
- Integrating a Linux application with Active Directory via LDAPS for centralized authentication
- PKI / certificate trust configuration so the host validates the DC's LDAPS certificate against an internal CA
- Least-privilege service-account design for directory binds
- ITSM service catalog and organizational modeling — departments, teams, roles, and help-topic routing
- SLA, escalation, and ticket-lifecycle workflow design with separation of duties
- Converging monitoring (Zabbix) and SIEM (Wazuh) telemetry into a single ticketing intake point
- Application, server, and directory security hardening
- HIPAA-aligned audit, accountability, and access-management controls

---

## Documentation

| Document | Description |
|---|---|
| [01-deployment.md](docs/01-deployment.md) | Debian host, Apache/PHP/MariaDB stack, and osTicket installation |
| [02-ad-authentication.md](docs/02-ad-authentication.md) | LDAPS authentication, `svc_osticket` service account, CA trust, and user/agent binding |
| [03-service-desk-structure.md](docs/03-service-desk-structure.md) | Departments, teams, agent roles, help topics, and ticket routing |
| [04-ticket-workflows.md](docs/04-ticket-workflows.md) | Ticket lifecycle, SLA plans, escalation rules, and canned responses |
| [05-monitoring-and-siem-integration.md](docs/05-monitoring-and-siem-integration.md) | Zabbix and Wazuh alert ingestion and the convergence model *(planned)* |
| [06-security-hardening.md](docs/06-security-hardening.md) | TLS on the web UI, install-dir removal, file permissions, scoped MariaDB grants, and SSH/host hardening |
| [07-backup-and-recovery.md](docs/07-backup-and-recovery.md) | Backup method (database, attachments, config), retention, and tested restore |

*Each document is published as its phase is documented.*

---

*Part of the [MedNet Enterprise Lab](../README.md) — Enterprise Healthcare IT Infrastructure & Security Operations Home Lab*
