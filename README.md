# MedNet Enterprise Lab

> **Enterprise Healthcare IT Infrastructure & Security Operations Home Lab**
> A fully integrated, multi-service enterprise environment simulating the infrastructure of a mid-size healthcare organization. Built to demonstrate real-world IT and security operations skills across networking, systems administration, monitoring, SIEM, automation, and endpoint management.

---

## Overview

MedNet Enterprise Lab is a virtualized enterprise environment built around a simulated healthcare organization operating under the domain **mednet.lab**. The lab is designed to reflect the complexity and security requirements of a real healthcare IT environment — not a basic home lab — with integrated services that communicate with and depend on one another the same way production infrastructure does.

The project covers the full stack of enterprise IT operations: identity and access management, file services, IT service management, infrastructure monitoring, security event management, endpoint administration, and automation. Every component is documented with the same standards expected in a professional environment.

This lab serves as the capstone portfolio project supporting a career transition into NOC analyst and network engineer roles, with a focus on healthcare IT environments where uptime, access control, and compliance are critical.

---

## Environment Summary

| Component | Platform | Role |
|---|---|---|
| Domain Controller | Windows Server | Active Directory, DNS, PKI/CA, LDAPS, GPO |
| File Server | Debian Linux | Samba shares, AD-integrated permissions |
| ITSM Platform | Debian Linux | osTicket, AD authentication, helpdesk workflows |
| Network Monitoring | Ubuntu Server | Zabbix, SNMP, alerting, osTicket integration |
| SIEM / HIDS | Rocky Linux 9 | Wazuh, log aggregation, active response |
| Workstation (Clinical) | Windows 11 Enterprise | Domain-joined endpoint, GPO applied, Wazuh agent |
| Workstation (Administrative) | Windows 10 LTSC 2021 | Domain-joined endpoint, GPO applied, Wazuh agent |
| Workstation (IT) | Ubuntu Desktop | Domain-joined Linux endpoint |
| Hypervisor | Oracle VirtualBox | All VMs hosted locally |

**Domain:** `mednet.lab`
**Domain Controller:** `dc01.mednet.lab`

---

## Architecture

The environment is built around Active Directory as the central identity provider. All services authenticate against the domain, and access to resources is governed by AD groups and GPOs organized around a realistic hospital organizational structure.

```
                             ┌───────────────────┐
                             │  dc01.mednet.lab  │
                             │  Active Directory │
                             │ DNS | PKI | LDAPS │
                             └─────────┬─────────┘
                                       │
       ┌───────────────┬───────────────┼───────────────┬───────────────┐
       │               │               │               │               │
┌──────┬─────┐  ┌──────┬─────┐  ┌──────┬─────┐  ┌──────┬─────┐  ┌──────┬─────┐
│  osTicket  │  │   Zabbix   │  │   Wazuh    │  │ File Server│  │Workstations│
│    ITSM    │  │  Monitor   │  │ SIEM/HIDS  │  │  Samba/AD  │  │ Win/Linux  │
└────────────┘  └────────────┘  └────────────┘  └────────────┘  └────────────┘
```

Service integrations:
- Zabbix automatically opens osTicket tickets on threshold alerts
- Wazuh agents deployed on all domain-joined endpoints, forwarding events to the SIEM
- osTicket authenticates users via LDAPS against Active Directory
- File server permissions enforced through AD security groups
- PowerShell automates the AD identity lifecycle (onboarding, offboarding, access reviews)
- Ansible enforces Linux baseline configuration and deploys monitoring/SIEM agents across the fleet
- Python and Bash drive operational health checks, API-driven reporting, and scheduled backups

---

## Repository Structure

```
MedNet-Enterprise-Lab/
├── README.md                          ← You are here
├── 01-MedNet-ActiveDirectory/         ← AD, GPOs, PKI, OU structure
├── 02-MedNet-FileServer/              ← Debian Samba, share structure, permissions
├── 03-MedNet-TicketingSystem/         ← ITSM platform, AD integration, workflows
├── 04-MedNet-NetworkMonitoring/       ← Monitoring, SNMP, dashboards, ticket integration
├── 05-MedNet-SIEM/                    ← SIEM, agents, active response, HIPAA rules
├── 06-MedNet-Workstations/            ← Endpoint configs, GPO applied, agents
└── 07-MedNet-Automation/              ← PowerShell, Ansible, Python/Bash automation
```

Each subfolder contains its own `README.md` with a service-specific overview and a `/docs` directory with detailed configuration documentation.

---

## Lab Design Principles

**Healthcare context is intentional.** The OU structure, security groups, GPOs, and workflows are modeled after a realistic hospital IT environment. This means strict access control, separation of duties between clinical and administrative staff, and security policies that reflect HIPAA-aligned thinking.

**Integration over isolation.** Every service is connected to at least one other. Zabbix feeds into osTicket. Wazuh watches the endpoints. AD governs access everywhere. The goal is to demonstrate that these tools work as a system, not just as individual installs.

**Security is layered.** The environment includes network-level monitoring (Zabbix), host-based intrusion detection (Wazuh), endpoint protection (Windows Defender ingested by Wazuh), and identity-enforced access control (Active Directory). This mirrors a real defense-in-depth approach.

**Documentation reflects real operations.** Docs are written as they would be in a professional environment — focused on design decisions, configurations, and validation rather than step-by-step tutorials.

---

## Skills Demonstrated

**Systems Administration**
- Windows Server administration and Active Directory management
- Linux administration across Debian, Ubuntu Server, and Rocky Linux 9
- GPO design and enforcement at scale
- Internal PKI and certificate authority management

**Networking & Infrastructure**
- DNS management within an enterprise domain
- SNMP monitoring and network device integration
- Threshold-based alerting and availability monitoring
- Samba file services with AD-integrated permissions

**Security Operations**
- SIEM deployment and log aggregation (Wazuh)
- Host-based intrusion detection and active response
- Security event correlation and alerting
- HIPAA-aligned policy and audit configuration

**IT Service Management**
- osTicket deployment and workflow configuration
- SLA design and enforcement
- Automated ticket creation from monitoring alerts
- Role-based access control for service desk operations

**Automation**
- AD identity lifecycle management with PowerShell (onboarding, offboarding, access reviews, stale-account auditing)
- Multi-distro configuration management and agent deployment with Ansible
- Operational health checks, API-driven reporting, and automated backups with Python and Bash
- Idempotent, logged, secret-safe automation scheduled via Task Scheduler and systemd timers/cron

**Documentation**
- Enterprise-grade technical documentation
- Network diagrams and infrastructure mapping
- Design-decision and validation-focused writing

---

LinkedIn: https://www.linkedin.com/in/samuel-j-adams-6a668a307/
