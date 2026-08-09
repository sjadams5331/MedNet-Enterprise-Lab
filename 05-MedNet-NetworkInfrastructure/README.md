# MedNet Network Infrastructure

> **pfSense-Based Network Segmentation and Perimeter Firewall for mednet.lab**
> A five-VLAN architecture built on pfSense, separating servers, clinical systems, administrative systems, IT/operations tooling, and a public-facing DMZ behind least-privilege firewall policy. Models the network-layer segmentation a HIPAA-adjacent healthcare environment would need to demonstrate, with every inter-VLAN path either explicitly justified or explicitly denied.

---

## Overview

This module implements network segmentation and perimeter firewall services for the MedNet Enterprise Lab using pfSense. The environment is divided into five VLANs that separate servers, clinical systems, administrative systems, IT/operations tooling, and a public-facing DMZ, with firewall policy enforcing least-privilege access between them.

The segmentation model is loosely informed by the kind of network boundaries a HIPAA-adjacent healthcare environment would need to demonstrate: clinical systems isolated from administrative and business systems, monitoring and SIEM tooling confined to an IT-only tier reachable only through a jump host, and the one system every department must reach (the ticketing portal) walled off in its own restricted segment rather than trusted broadly.

This module builds directly on the identity, file, and ticketing services established in Modules 01 through 03, migrating each of those hosts off the original flat network and into the segmented environment defined here.

---

## Environment Details

| Setting | Value |
|---|---|
| Hostname | `pfsense.mednet.lab` |
| Platform | pfSense CE 2.8.1-RELEASE |
| Role | Perimeter firewall and inter-VLAN router |
| Interfaces | 1 WAN, 5 internal (LAN + OPT1-4) |
| VLANs | 5 (Servers, Clinical, Admin, IT/Ops, DMZ) |
| WAN | VirtualBox NAT, dynamic (`10.0.2.0/24`) |

---

## Prerequisites

| Dependency | Details |
|---|---|
| Active Directory | `dc01.mednet.lab`, reachable and hosting DNS for `mednet.lab` |
| File Server | `mednet-fs01.mednet.lab`, domain-joined and serving department shares |
| ITSM Platform | `itsm01.mednet.lab`, LDAPS-integrated osTicket deployment |
| Hypervisor | VirtualBox on the Arch Linux host, with support for six NICs per VM (one WAN, five internal) |
| pfSense Installer | pfSense CE 2.8.1-RELEASE installation media |

---

## VLAN Architecture at a Glance

| VLAN | Subnet | Purpose | Key Hosts |
|---|---|---|---|
| 10, Servers | `10.10.10.0/24` | Domain services | Domain Controller, File Server |
| 20, Clinical | `10.10.20.0/24` | Clinical workstations | WS-CLIN-01 |
| 30, Admin | `10.10.30.0/24` | Administrative workstations | WS-ADMIN-01 |
| 40, IT/Ops | `10.10.40.0/24` | IT staff and monitoring | WS-IT-01, Zabbix, Wazuh |
| 50, DMZ | `10.10.50.0/24` | Public-facing services | osTicket |

Each VLAN is realized as a dedicated VirtualBox Internal Network attached to its own pfSense interface, rather than a single trunked NIC carrying 802.1Q tags. This keeps each segment's broadcast domain genuinely isolated at the hypervisor level, with pfSense handling routing and access control between them.

---

## Architecture

```
                                Internet
                                   │
                                (WAN, NAT)
                                   │
                             ┌───────────┐
                             │  pfSense  │
                             │  Firewall │
                             └─────┬─────┘
        ┌────────────┬────────────┼────────────┬────────────┐
        │            │             │            │            │
     VLAN 10       VLAN 20      VLAN 30      VLAN 40       VLAN 50
     Servers       Clinical      Admin        IT/Ops         DMZ
   10.10.10.0/24  10.10.20.0/24 10.10.30.0/24 10.10.40.0/24 10.10.50.0/24
   DC, File Srv    WS-CLIN-01   WS-ADMIN-01   WS-IT-01,     osTicket
                                              Zabbix, Wazuh
```

IT/Ops (VLAN 40) is the one segment with broad reach into every other VLAN, reflecting its role as the management and monitoring plane. Clinical and Admin cannot reach each other. The DMZ can be reached by every department on port 443, but has exactly one narrow, defined reason to initiate a connection elsewhere (LDAPS to the domain controller).

---

## Key Design Decisions

- **Destination-scoped rules over broad allow rules.** Clinical and Admin workstations are permitted to reach specific, named infrastructure (the domain controller, the file server, the ticketing portal) rather than the internal network generally, with a private-address-space block closing off everything else internal by default.
- **Clinical and Admin cannot reach each other.** A deliberate isolation boundary between clinical and business systems, enforced by firewall policy rather than left to topology.
- **IT/Ops (VLAN 40) retains broad internal reach**, reflecting its role as the management and monitoring tier rather than a client segment.
- **The DMZ (VLAN 50) is broadly reachable inbound and tightly restricted outbound**, the standard shape of a DMZ: every department can reach the ticketing portal, but the portal itself has exactly one narrow, defined reason to initiate a connection elsewhere.
- **Administration happens through a jump host, not a flat path from the hypervisor's host machine.** VirtualBox Internal Networks are VM-to-VM only by design; WS-IT-01 serves as the point of access into the segmented environment, mirroring how a real network restricts administrative reach through a bastion host rather than direct access from every workstation.

---

## Repository Structure

```
05-MedNet-NetworkInfrastructure/
├── README.md                                  ← You are here
├── 01-network-segmentation-design.md          ← VLAN architecture, addressing, and policy rationale
├── 02-firewall-and-routing-configuration.md   ← pfSense interface setup and firewall rule logic
├── 03-vlan-and-inter-vlan-routing.md          ← Host migration, routing verification, and troubleshooting
├── 04-security-hardening.md                   ← Hardening decisions, implemented and deferred
└── screenshots/
```

---

## Skills Demonstrated

- VLAN-based network segmentation design and least-privilege inter-VLAN access policy
- pfSense firewall rule engineering: destination-scoped allow rules, alias-based blocking, and rule-order reasoning
- Host migration planning and execution across a live, multi-VLAN environment
- Inter-VLAN routing verification using TTL analysis and pfSense's state table
- Multi-homed host routing troubleshooting, both Windows (`RouteMetric`) and Linux (NetworkManager route-metric)
- Perimeter firewall hardening: administrative access controls, SSH key-only authentication, and a CA-signed GUI certificate
- Design decisions mapped to CCNA-aligned networking concepts and real-world HIPAA-adjacent segmentation practice

---

## Documentation

| Document | Covers |
|---|---|
| [01-network-segmentation-design.md](01-network-segmentation-design.md) | VLAN architecture and subnet design rationale |
| [02-firewall-and-routing-configuration.md](02-firewall-and-routing-configuration.md) | pfSense interface setup and firewall rule logic |
| [03-vlan-and-inter-vlan-routing.md](03-vlan-and-inter-vlan-routing.md) | Host migration, inter-VLAN routing verification, and a detailed troubleshooting account |
| [04-security-hardening.md](04-security-hardening.md) | Hardening decisions, both implemented and deliberately deferred |

---

## Status

Segmentation, host migration, firewall policy, and hardening are complete and verified end to end. All five VLANs are live, every planned host has been migrated off the original flat network, inter-VLAN routing has been confirmed via TTL analysis, and the DMZ's LDAPS-only outbound path has been validated against a live authentication test. The lateral-movement block between Clinical and Admin, LAN outbound narrowing, and pfSense's own administrative hardening (HTTPS-only GUI, CA-signed certificate, SSH key-only access, login-protection thresholds) are all live.

Four items remain intentionally deferred rather than built prematurely: port-level scoping on IT/Ops (OPT3), which depends on real Zabbix/Wazuh agent traffic that doesn't exist until Modules 04 and 06 are built; two-factor authentication for the pfSense admin account; renaming the default `admin` account; and importing the internal CA into the two non-domain-joined hosts' trust stores. Each is noted plainly in [04-security-hardening.md](04-security-hardening.md) along with the reasoning behind deferring it.

---

## Related Documents

| Document | Description |
|---|---|
| [../docs/architecture-overview.md](../docs/architecture-overview.md) | Capstone document tying all MedNet modules together |
| [../04-MedNet-NetworkMonitoring/README.md](../04-MedNet-NetworkMonitoring/README.md) | Zabbix deployment (VLAN 40) |
| [../06-MedNet-SIEM/README.md](../06-MedNet-SIEM/README.md) | Wazuh deployment (VLAN 40) |

---

*Part of the [MedNet Enterprise Lab](../README.md), an Enterprise Healthcare IT Infrastructure & Security Operations home lab.*
