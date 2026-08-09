# 04 – Network Monitoring (Zabbix)

Centralized host-level monitoring for the MedNet environment, built on Zabbix 7.0 LTS. Covers the six core endpoints spanning the Servers, Clinical, Admin, IT/Ops, and DMZ VLANs, plus self-monitoring of the Zabbix server itself.

## Documentation in This Module

| Doc | Covers |
|---|---|
| [01-architecture-and-scope.md](./01-architecture-and-scope.md) | Design intent, monitoring scope, host inventory, the platform's origin as a repurposed VM upgraded off an EOL version |
| [02-host-and-agent-configuration.md](./02-host-and-agent-configuration.md) | Per-host setup, passive vs. active check strategy, the Agent vs. Agent 2 distinction on Windows |
| [03-network-and-firewall-configuration.md](./03-network-and-firewall-configuration.md) | VLAN-to-interface mapping and the pfSense rules required for cross-segment monitoring traffic |
| [04-triggers-and-alerting.md](./04-triggers-and-alerting.md) | Current alerting state (vendor defaults only) and what's still needed |
| [05-validation-and-troubleshooting.md](./05-validation-and-troubleshooting.md) | How each host was validated, and a full account of the real issues hit along the way |

## Environment Summary

| Host | Role | VLAN | Check type |
|---|---|---|---|
| `dc01` | Domain Controller | 10 – Servers | Passive |
| `mednet-fs01` | File Server | 10 – Servers | Passive |
| `itsm01` | Helpdesk (osTicket) | 50 – DMZ | Passive |
| `WS-IT-01` | IT Workstation | 40 – IT/Ops | Passive |
| `WS-CLIN-01` | Clinical Workstation | 20 – Clinical | Active (DHCP) |
| `WS-ADMIN-01` | Admin Workstation | 30 – Admin | Active (DHCP) |
| `mon01` | Zabbix Server (self) | 40 – IT/Ops | Passive |

## Status

All seven hosts (six endpoints plus self-monitoring) are actively reporting. This module demonstrates monitoring deployment across a segmented, firewall-enforced network, including the real complications that introduces, rather than a flat, permissive lab network.

**Still open:**
- A misconfigured pfSense rule on the Clinical VLAN interface needs a final fix/confirmation
- Alerting is vendor-default only; no custom triggers or notification actions are configured yet
- Agent-server traffic is unencrypted and the Zabbix frontend runs over plain HTTP; acceptable for an internal, firewalled lab, but a gap worth closing for a security-focused portfolio narrative

See `04-triggers-and-alerting.md` and `05-validation-and-troubleshooting.md` for the reasoning behind these being left open rather than silently resolved.
