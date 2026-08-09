# Architecture and Monitoring Scope

## Overview

This module implements centralized monitoring for the MedNet environment using Zabbix 7.0 LTS, positioned as a real-time monitoring and alerting layer rather than a full observability or log-aggregation platform. The goal was to establish reliable visibility into the six endpoints that make up the environment's core infrastructure, clinical, admin, IT, and DMZ tiers, consistent with how a small enterprise NOC would scope an initial monitoring rollout: host availability and resource health first, deeper application-level instrumentation later.

## Platform Origin

The Zabbix server was not built from scratch for this module. It's a VirtualBox VM (`ZabbixLab`, referenced throughout this documentation and in Zabbix itself as `mon01`) that was originally provisioned for an earlier, unrelated lab that was never completed. Repurposing an existing VM rather than provisioning a new one is a realistic decision a real NOC operator might make when infrastructure is available, and it's documented here rather than hidden, consistent with how `dc01`'s real hostname (`WIN-1UKKKVRDHPB`) is preserved rather than papered over.

That reuse came with a real consequence: the installed Zabbix version was 6.4, which reached end of active support on June 30, 2024. Before onboarding any hosts, the server was upgraded in place to **Zabbix 7.0.29 LTS** (supported through June 2029), following a VM snapshot and a full database/config backup. The upgrade included resolving a stale package repository pointer, a database schema migration on first start, and a local self-monitoring hostname mismatch left over from the prior installation. This is covered in detail in `05-validation-and-troubleshooting.md`.

## Monitoring Scope

Monitoring was scoped to answer three operational questions, consistent with the environment's overall design philosophy:

- Are the six core hosts reachable?
- Are they approaching resource exhaustion (CPU, memory, disk)?
- Is the underlying agent/monitoring pipeline itself healthy?

Zabbix is not used here as a log aggregation or SIEM platform; that role belongs to Wazuh (Module 06). Deep application-level monitoring (e.g., osTicket queue depth, AD replication health) was intentionally left out of this initial rollout in favor of establishing solid host-level coverage first.

## Monitored Hosts

| Host | Role | OS | VLAN / Subnet | IP |
|---|---|---|---|---|
| `dc01` (`WIN-1UKKKVRDHPB`) | Active Directory Domain Controller | Windows Server | 10 – Servers | 10.10.10.10 |
| `mednet-fs01` | File Server (Samba) | Debian 12 | 10 – Servers | 10.10.10.20 |
| `itsm01` (`osticket01`) | Helpdesk (osTicket) | Debian 12 | 50 – DMZ | 10.10.50.120 |
| `WS-IT-01` | IT Workstation | Ubuntu 24.04 | 40 – IT/Ops | 10.10.40.10 |
| `WS-CLIN-01` | Clinical Workstation | Windows | 20 – Clinical | DHCP |
| `WS-ADMIN-01` | Admin Workstation | Windows | 30 – Admin | DHCP |
| `mon01` (self) | Zabbix Server | Ubuntu 22.04 | 40 – IT/Ops | 10.10.40.30 |

All seven hosts (six monitored endpoints plus self-monitoring) are actively reporting as of this writing. Full per-host configuration, including the split between passive and active checks, is documented in `02-host-and-agent-configuration.md`.

## Design Principles

- **Segmentation-aware monitoring.** Unlike the earlier, unfinished lab this platform was repurposed from, MedNet's network is fully VLAN-segmented behind pfSense. Every cross-VLAN monitoring path required an explicit, scoped firewall rule rather than relying on a flat network's implicit reachability. This is a meaningfully more realistic and more complex monitoring design than the prior lab attempted, and it's documented as its own module (`03-network-and-firewall-configuration.md`) rather than an afterthought.
- **DHCP-aware check strategy.** The two clinical/admin workstations receive DHCP addresses rather than static ones, which makes IP-based polling unreliable over time. Rather than assign static reservations purely to simplify monitoring, these two hosts use Zabbix's active-check model instead, where the agent initiates contact with the server rather than the reverse. See `02-host-and-agent-configuration.md` for the reasoning and the real issues this introduced.
- **Standard templates, minimal customization.** All hosts use Zabbix's vendor-supplied Windows/Linux agent templates without heavy modification, to preserve supportability and demonstrate practical use of widely-deployed monitoring standards rather than bespoke configuration that would be harder to maintain or explain.

## Known Gaps

This module is functionally complete for host-level availability and resource monitoring, but two areas remain intentionally unfinished as of this writing:

- **Alerting is currently vendor-default only.** No custom triggers, actions, or notification routing have been configured. See `04-triggers-and-alerting.md`.
- **Agent-server communication is unencrypted** (`Agent encryption: None` across all hosts), and the Zabbix frontend is served over plain HTTP rather than HTTPS. This is a reasonable state for an internal, firewall-segmented lab, but is called out here explicitly rather than left unaddressed, since it's the kind of gap a real hardening pass would need to close before this platform could be considered production-representative.
