# Network Segmentation Design

## Overview

This document defines the VLAN architecture, IP addressing plan, and inter-VLAN access policy for the MedNet environment. It establishes network-layer separation between PHI-adjacent clinical systems, general administrative/business functions, IT and monitoring infrastructure, and the service desk platform — consistent with the "minimum necessary" and access-control principles underlying HIPAA's Security Rule (45 CFR §164.312).

This design is the target state for the environment. Migration of existing hosts from the flat `192.168.56.0/24` network into the VLANs defined here is covered separately in [`03-vlan-and-inter-vlan-routing.md`](03-vlan-and-inter-vlan-routing.md).

## Design Goals

- Isolate PHI-adjacent clinical traffic from general administrative traffic at layer 3, not just by policy
- Give IT/monitoring a controlled management plane that can reach every segment, since Zabbix (Module 04) and Wazuh (Module 06) both depend on this
- Constrain the ITSM platform — which is not domain-joined and authenticates over LDAPS only — to the minimum server access it actually needs
- Use addressing that's easy to reason about: the VLAN ID is visible directly in the subnet
- Default-deny between VLANs; every permitted path is explicit and has a stated reason

## VLAN Architecture

| VLAN ID | Name | Subnet | Gateway (pfSense) | Purpose |
|---|---|---|---|---|
| 10 | Servers | `10.10.10.0/24` | `10.10.10.1` | Domain controller, file server, and future core infrastructure |
| 20 | Clinical | `10.10.20.0/24` | `10.10.20.1` | Clinical workstations and future clinical-adjacent devices |
| 30 | Admin | `10.10.30.0/24` | `10.10.30.1` | Administrative / business-office workstations |
| 40 | IT / Ops | `10.10.40.0/24` | `10.10.40.1` | IT staff workstations, Zabbix, Wazuh — the management plane |
| 50 | DMZ (ITSM) | `10.10.50.0/24` | `10.10.50.1` | osTicket — semi-trusted, not domain-joined |

The existing `192.168.56.0/24` network becomes pfSense's WAN-side uplink only; no production hosts remain on it after the migration in doc 03.

## IP Addressing Plan

| VLAN | Host | Address |
|---|---|---|
| 10 — Servers | `dc01.mednet.lab` (`WIN-1UKKKVRDHPB`) | `10.10.10.10` |
| 10 — Servers | `mednet-fs01.mednet.lab` | `10.10.10.20` |
| 10 — Servers | *reserved for future infrastructure* | `10.10.10.30–49` |
| 20 — Clinical | `WS-CLIN-01` | `10.10.20.10` |
| 20 — Clinical | DHCP scope (future clinical devices) | `10.10.20.100–200` |
| 30 — Admin | `WS-ADMIN-01` | `10.10.30.10` |
| 30 — Admin | DHCP scope | `10.10.30.100–200` |
| 40 — IT/Ops | `WS-IT-01` | `10.10.40.10` |
| 40 — IT/Ops | `mon01.mednet.lab` (Zabbix) | `10.10.40.30` |
| 40 — IT/Ops | `siem01.mednet.lab` (Wazuh) | `10.10.40.50` |
| 50 — DMZ | `itsm01.mednet.lab` | `10.10.50.120` |

`.1` in each subnet is reserved for the pfSense interface acting as that VLAN's gateway. `.2–.9` are reserved for future static infrastructure.

## Inter-VLAN Access Policy

| Source | Destination | Allowed | Rationale |
|---|---|---|---|
| Clinical (20) | Servers (10) | DNS 53, Kerberos 88, LDAP 389, SMB 445, AD RPC | AD authentication and file share access for clinical workflows |
| Clinical (20) | DMZ (50) | HTTPS 443 | Submitting and viewing support tickets |
| Clinical (20) | Admin (30) | Deny | No business justification; keeps PHI-adjacent segment isolated |
| Clinical (20) | IT/Ops (40) | Deny (inbound-only from IT/Ops) | Management plane reaches in; clinical never initiates outbound to it |
| Admin (30) | Servers (10) | DNS, Kerberos, LDAP, SMB | Same AD/file access as Clinical, separate business function |
| Admin (30) | DMZ (50) | HTTPS 443 | Ticketing access |
| Admin (30) | Clinical (20) | Deny | Segmentation between business and clinical functions |
| IT/Ops (40) | All VLANs | SSH 22, RDP 3389, SNMP, Zabbix agent 10050/10051, Wazuh agent 1514/1515, ICMP | Centralized management and monitoring needs reach into every segment |
| DMZ (50) | Servers (10), DC only | LDAPS 636 | App-level auth only — ITSM is not domain-joined, so its blast radius into Servers stays minimal |
| DMZ (50) | Any other VLAN | Deny | Ticketing platform has no reason to initiate connections elsewhere |
| Any VLAN | WAN | Per-host update/patch needs | Standard outbound internet access via pfSense NAT |
| *(unlisted)* | *(unlisted)* | Deny | Default-deny inter-VLAN posture |

This table defines intent. The actual pfSense rule implementation is covered in [`02-firewall-and-routing-configuration.md`](02-firewall-and-routing-configuration.md).

## Design Constraints & Assumptions

- **No 802.1Q trunking.** This build gives pfSense one dedicated LAN interface per VLAN (each backed by a separate VirtualBox internal network) rather than a single trunked link with tagged sub-interfaces. VirtualBox internal networks isolate broadcast domains without requiring trunk-aware NICs, so segmentation, routing, and firewall behavior are functionally equivalent to a trunked deployment — the difference is purely in virtual link count on the router. VLAN IDs are retained in naming and documentation to mirror how this design would map onto a trunked switchport in a production or CCNA-style environment.
- **WAN placement.** pfSense's WAN interface stays on the existing `192.168.56.0/24` segment for outbound internet access (updates, package repos, etc.).
- **Migration is out of scope here.** This document defines the target addressing and policy only. Moving the DC, file server, and workstations into their new subnets — and validating AD/DNS/name resolution survives the move — happens in doc 03.

## Related Documents

| Document | Description |
|---|---|
| [02-firewall-and-routing-configuration.md](02-firewall-and-routing-configuration.md) | pfSense rule implementation for the policy defined above |
| [03-vlan-and-inter-vlan-routing.md](03-vlan-and-inter-vlan-routing.md) | VLAN interface build-out and host migration |
| [04-security-hardening.md](04-security-hardening.md) | pfSense hardening pass |
| [../../01-MedNet-ActiveDirectory/README.md](../../01-MedNet-ActiveDirectory/README.md) | DC being migrated into the Servers VLAN |
| [../../02-MedNet-FileServer/README.md](../../02-MedNet-FileServer/README.md) | File server being migrated into the Servers VLAN |

---
*Part of the [MedNet Enterprise Lab](../README.md) · Module 05 — NetworkInfrastructure*
