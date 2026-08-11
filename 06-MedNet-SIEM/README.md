# 06 – SIEM (Wazuh)

Centralized security event monitoring, log aggregation, and file integrity monitoring for the MedNet environment, built on Wazuh 4.14.4. Covers manager deployment and agent rollout across all six core endpoints spanning the Servers, Clinical, Admin, IT/Ops, and DMZ VLANs.

---

## Documentation in This Module

| Doc | Covers |
|---|---|
| [01-architecture-and-deployment.md](./01-architecture-and-deployment.md) | Platform choice, manager deployment on Rocky Linux, host firewall design, and the DNS issues resolved during initial setup |
| [02-log-source-and-agent-integration.md](./02-log-source-and-agent-integration.md) | Per-host agent rollout, the pfSense and firewalld rules required for each VLAN, and the real issues hit along the way |
| 03-detection-rules-and-alerting.md | Not yet written. Custom detection rules and alerting, beyond the vendor-default ruleset currently active |
| 04-security-hardening.md | Not yet written. Dashboard access control, TLS certificate replacement, and agent authentication hardening |

## Environment Summary

| Host | Role | VLAN | IP | OS | Agent name |
|---|---|---|---|---|---|
| `siem01` | Wazuh Manager (self) | 40 – IT/Ops | 10.10.40.50 | Rocky Linux 9 | n/a, manager |
| `dc01` (`WIN-1UKKKVRDHPB`) | Domain Controller | 10 – Servers | 10.10.10.10 | Windows Server 2025 | `dc01` |
| `mednet-fs01` | File Server | 10 – Servers | 10.10.10.20 | Debian 12 | `fs01` |
| `itsm01` (`osticket01`) | Helpdesk (osTicket) | 50 – DMZ | 10.10.50.120 | Debian 12 | `itsm01` |
| `WS-IT-01` | IT Workstation | 40 – IT/Ops | 10.10.40.10 | Ubuntu 24.04 | `WS-IT-01` |
| `WS-CLIN-01` | Clinical Workstation | 20 – Clinical | 10.10.20.100 | Windows 11 Enterprise | `WS-CLIN-01` |
| `WS-ADMIN-01` | Admin Workstation | 30 – Admin | 10.10.30.100 | Windows 10 Enterprise LTSC 2021 | `WS-ADMIN-01` |

---

## Status

All six endpoints are actively reporting to the manager, confirmed through the Wazuh dashboard's Endpoints view. Manager deployment and agent connectivity are complete. Detection content is not.

**Done:**
- Wazuh manager deployed on Rocky Linux and reachable over HTTPS
- All six agents installed, enrolled, and reporting Active status
- Cross-VLAN firewall rules in place on pfSense for every non-local segment, plus the manager's own host-level firewalld rules

**Still open:**
- Detection is currently vendor-default only. Every agent is running the default File Integrity Monitoring scan and a Security Configuration Assessment policy scoped to its OS (for example, `cis_win2025.yml` on `dc01`), but no custom rules, decoders, or alerting actions have been configured. See `03-detection-rules-and-alerting.md`.
- The manager is still serving its default self-signed certificate rather than one issued by `MedNet-RootCA`, despite that CA already being distributed to other Linux service VMs in the environment for exactly this purpose. See `04-security-hardening.md`.
- A stray, incorrectly-scoped pfSense rule on the OPT1 interface was flagged as an open item during Module 04's work (see [`04-MedNet-NetworkMonitoring/03-network-and-firewall-configuration.md`](../04-MedNet-NetworkMonitoring/03-network-and-firewall-configuration.md)) and was not independently reconfirmed as fixed during this module.

See `02-log-source-and-agent-integration.md` for a full account of what it actually took to get here. Most of the real troubleshooting in this module was network-layer, not Wazuh-specific.

---

*Part of the [MedNet Enterprise Lab](../README.md), an Enterprise Healthcare IT Infrastructure & Security Operations home lab.*
