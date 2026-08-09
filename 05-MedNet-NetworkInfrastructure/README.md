# MedNet Network Infrastructure

## Overview

This module implements network segmentation and perimeter firewall services for the MedNet Enterprise Lab using pfSense. The environment is divided into five VLANs that separate servers, clinical systems, administrative systems, IT/operations tooling, and a public-facing DMZ, with firewall policy enforcing least-privilege access between them.

The segmentation model is loosely informed by the kind of network boundaries a HIPAA-adjacent healthcare environment would need to demonstrate: clinical systems isolated from administrative and business systems, monitoring and SIEM tooling confined to an IT-only tier reachable only through a jump host, and the one system every department must reach (the ticketing portal) walled off in its own restricted segment rather than trusted broadly.

## Architecture at a Glance

| VLAN | Subnet | Purpose | Key Hosts |
|---|---|---|---|
| 10, Servers | 10.10.10.0/24 | Domain services | Domain Controller, File Server |
| 20, Clinical | 10.10.20.0/24 | Clinical workstations | WS-CLIN-01 |
| 30, Admin | 10.10.30.0/24 | Administrative workstations | WS-ADMIN-01 |
| 40, IT/Ops | 10.10.40.0/24 | IT staff and monitoring | WS-IT-01, Zabbix, Wazuh |
| 50, DMZ | 10.10.50.0/24 | Public-facing services | osTicket |

Each VLAN is realized as a dedicated VirtualBox Internal Network attached to its own pfSense interface, rather than a single trunked NIC carrying 802.1Q tags. This keeps each segment's broadcast domain genuinely isolated at the hypervisor level, with pfSense handling routing and access control between them.

## Key Design Decisions

- **Destination-scoped rules over broad allow rules.** Clinical and Admin workstations are permitted to reach specific, named infrastructure (the domain controller, the file server, the ticketing portal) rather than the internal network generally, with a private-address-space block closing off everything else internal by default.
- **Clinical and Admin cannot reach each other.** A deliberate isolation boundary between clinical and business systems, enforced by firewall policy rather than left to topology.
- **IT/Ops (VLAN 40) retains broad internal reach**, reflecting its role as the management and monitoring tier rather than a client segment.
- **The DMZ (VLAN 50) is broadly reachable inbound and tightly restricted outbound**, the standard shape of a DMZ: every department can reach the ticketing portal, but the portal itself has exactly one narrow, defined reason to initiate a connection elsewhere.
- **Administration happens through a jump host, not a flat path from the hypervisor's host machine.** VirtualBox Internal Networks are VM-to-VM only by design; WS-IT-01 serves as the point of access into the segmented environment, mirroring how a real network restricts administrative reach through a bastion host rather than direct access from every workstation.

## Documentation Index

| Document | Covers |
|---|---|
| [01-network-segmentation-design.md](01-network-segmentation-design.md) | VLAN architecture and subnet design rationale |
| [02-firewall-and-routing-configuration.md](02-firewall-and-routing-configuration.md) | pfSense interface setup and firewall rule logic |
| [03-vlan-and-inter-vlan-routing.md](03-vlan-and-inter-vlan-routing.md) | Host migration, inter-VLAN routing verification, and a detailed troubleshooting account |
| [04-security-hardening.md](04-security-hardening.md) | Hardening decisions, both implemented and deliberately deferred |

## Status

Segmentation, host migration, and core firewall policy are complete and verified end to end, including inter-VLAN routing (TTL-confirmed) and DMZ-scoped authentication traffic. Security hardening is partially complete: lateral-movement blocking between Clinical and Admin, and the restrictive DMZ posture, are both live. Outbound narrowing on the Servers VLAN and port-level scoping on IT/Ops (pending monitoring agent deployment in later modules) remain open, and are documented plainly as deferred rather than presented as finished.

## Related Documents

| Document | Description |
|---|---|
| [../docs/architecture-overview.md](../docs/architecture-overview.md) | Capstone document tying all MedNet modules together |
| [../04-MedNet-NetworkMonitoring/README.md](../04-MedNet-NetworkMonitoring/README.md) | Zabbix deployment (VLAN 40) |
| [../06-MedNet-SIEM/README.md](../06-MedNet-SIEM/README.md) | Wazuh deployment (VLAN 40) |

*Part of the MedNet Enterprise Lab*
