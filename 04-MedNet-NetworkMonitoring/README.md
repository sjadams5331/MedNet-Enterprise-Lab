# MedNet Zabbix
 
> **Infrastructure Monitoring & Alerting for mednet.lab**
> Zabbix deployment providing full-stack infrastructure monitoring across the MedNet healthcare environment. All servers and workstations are monitored via Zabbix agents, with threshold-based alerting feeding directly into osTicket for automated ticket creation and SLA-bound incident tracking.
 
---
 
## Overview
 
`mon01.mednet.lab` runs Zabbix on Ubuntu Server and serves as the NOC-layer monitoring platform for the entire MedNet environment. Every host in the domain — servers, workstations, and the domain controller itself — reports to Zabbix via an installed agent. Performance metrics, service availability, and system health are tracked continuously, with alerts configured to reflect the uptime expectations of a healthcare environment where system availability directly impacts operations.
 
Zabbix operates at a different layer than Wazuh. Where Wazuh handles security events and log analysis, Zabbix handles infrastructure performance and availability — CPU, memory, disk, network throughput, and service uptime. Together they provide both the NOC and SOC visibility layers expected in a mature enterprise environment.
 
The most operationally significant feature of this deployment is the automated pipeline between Zabbix and osTicket. When a monitored host breaches a defined threshold, Zabbix automatically opens a ticket in the NOC / Infrastructure queue — no manual handoff required.
 
---
 
## Server Details
 
| Property | Value |
|---|---|
| Hostname | `mon01.mednet.lab` |
| Operating System | Ubuntu Server |
| Zabbix Version | Zabbix 6.x LTS |
| Database | MySQL |
| Web Frontend | Apache |
| Agent Port | 10050 |
| Server Port | 10051 |
 
---
 
## Architecture
 
```
                    ┌─────────────────────┐
                    │   mon01.mednet.lab  │
                    │   Zabbix Server     │
                    │   Ubuntu Server     │
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼─────────────────────┐
          │           │        │        │             │
    ┌─────┴────┐ ┌────┴───┐ ┌─┴──────┐ ┌┴─────────┐ ┌┴──────────┐
    │dc01      │ │itsm01  │ │siem01  │ │fs01      │ │Workstations│
    │AD Server │ │osTicket│ │Wazuh   │ │File Server│ │Win/Linux  │
    │Zabbix Agt│ │Zabbix  │ │Zabbix  │ │Zabbix Agt│ │Zabbix Agt │
    └──────────┘ │Agent   │ │Agent   │ └──────────┘ └───────────┘
                 └────────┘ └────────┘
                               │
                    ┌──────────▼──────────┐
                    │   itsm01.mednet.lab │
                    │   osTicket API      │
                    │   Auto ticket on    │
                    │   threshold breach  │
                    └─────────────────────┘
```
 
---
 
## Monitored Hosts
 
All hosts in the `mednet.lab` domain are monitored via the Zabbix agent. Agents are deployed on each VM and report back to `mon01.mednet.lab` on port 10050.
 
| Host | OS | Role | Template Applied |
|---|---|---|---|
| `dc01.mednet.lab` | Windows Server | Domain Controller | Windows by Zabbix Agent |
| `itsm01.mednet.lab` | Debian 12 | osTicket ITSM | Linux by Zabbix Agent |
| `siem01.mednet.lab` | Rocky Linux 9 | Wazuh SIEM | Linux by Zabbix Agent |
| `fs01.mednet.lab` | Debian | File Server | Linux by Zabbix Agent |
| `win10-ws.mednet.lab` | Windows 10 | Workstation | Windows by Zabbix Agent |
| `win11-ws.mednet.lab` | Windows 11 | Workstation | Windows by Zabbix Agent |
| `ubuntu-ws.mednet.lab` | Ubuntu Desktop | Workstation | Linux by Zabbix Agent |
 
---
 
## Monitored Metrics
 
### All Hosts
- CPU utilization
- Memory usage
- Disk space usage per partition
- Disk read/write throughput
- Network interface traffic (inbound/outbound)
- System uptime
- Agent availability
### Windows Hosts (dc01, win10-ws, win11-ws)
- Windows service status (critical services)
- Event log error monitoring
- Active network connections
- Logged-in user sessions
### Linux Hosts (itsm01, siem01, fs01, ubuntu-ws)
- Running process count
- System load average
- Open file descriptors
- SSH service availability
- Apache/MariaDB service status (where applicable)
### Domain Controller (dc01.mednet.lab)
- Active Directory service status (NTDS, DNS, Netlogon, KDC)
- DNS query response time
- LDAP service availability on port 636

---
 
## Alert Thresholds
 
Thresholds are configured to reflect the availability expectations of a healthcare IT environment. Alerts are tiered by severity to avoid alert fatigue while ensuring critical conditions are never missed.
 
| Metric | Warning Threshold | Critical Threshold |
|---|---|---|
| CPU utilization | 80% for 5 min | 95% for 3 min |
| Memory usage | 85% | 95% |
| Disk space usage | 80% | 90% |
| Agent unavailable | — | 3 minutes |
| Windows service down | — | 1 check |
| AD services down | — | 1 check |
| Apache/MariaDB down | — | 1 check |
 
---
 
## osTicket Integration
 
When a Zabbix trigger fires at Warning severity or above, an automated action executes a script on `mon01.mednet.lab` that sends a POST request to the osTicket REST API. This creates a ticket in the NOC / Infrastructure department with full alert context pre-populated.
 
### Workflow
 
```
Threshold breached → Zabbix trigger fires → Action executes script
→ POST to osTicket API → Ticket created in NOC queue → SLA clock starts
→ Trigger resolves → Recovery action appends resolution note to ticket
```
 
### Ticket Details
 
| Property | Value |
|---|---|
| Routed Department | NOC / Infrastructure |
| Help Topic | Infrastructure Alert (Zabbix) |
| Default Priority | High |
| Subject Format | `[Zabbix Alert] {HOST.NAME} — {TRIGGER.NAME}` |
| Body Content | Host, trigger name, severity, current value, timestamp |
 
---
 
## Dashboards
 
Department-scoped dashboards are configured to provide relevant views for different operational roles. Rather than a single global view, each dashboard is focused on what a specific team needs to see.
 
| Dashboard | Audience | Content |
|---|---|---|
| NOC Overview | Infrastructure team | All hosts, active alerts, uptime summary |
| Server Health | Systems administrators | Per-server CPU, memory, disk trends |
| Workstation Status | Help desk | Workstation availability and agent status |
| Domain Controller | IT leadership | AD services, DNS health, LDAP availability |
  
---
 
## Zabbix Host — Self Monitoring
 
`mon01.mednet.lab` is added as a monitored host in Zabbix, monitoring the monitoring platform itself. This ensures that if the Zabbix server experiences resource pressure it is detected and alerted on like any other host.
 
---
 
## Integration Points
 
| Service | Integration |
|---|---|
| `dc01.mednet.lab` | Agent monitoring of AD, DNS, and LDAP services |
| `itsm01.mednet.lab` | Agent monitoring plus receives automated alert tickets via API |
| `siem01.mednet.lab` | Agent monitoring of Wazuh host resources |
| `fs01.mednet.lab` | Agent monitoring of file server health and disk usage |
| Workstations | Agent monitoring of all domain-joined endpoints |
 
---
 
## Validation
 
- All agents confirmed active and reporting — verified in Zabbix host inventory
- Threshold alerts confirmed firing correctly under simulated load conditions
- osTicket ticket creation confirmed end-to-end from Zabbix trigger to NOC queue
- Recovery actions confirmed appending resolution notes to existing tickets
- Department-scoped dashboards verified displaying correct host subsets
- AD service monitoring confirmed detecting simulated service disruption on dc01
---
 
## Documentation
 
| Document | Description |
|---|---|
| [01-environment-and-setup.md](docs/01-environment-and-setup.md) | Server details, installation, agent deployment across all hosts |
| [02-monitoring-configuration.md](docs/02-monitoring-configuration.md) | Host setup, templates, metrics, and threshold configuration |
| [03-alerting-and-dashboards.md](docs/03-alerting-and-dashboards.md) | Alert actions, dashboard configuration, department-scoped views |
| [04-osticket-integration.md](docs/04-osticket-integration.md) | Automated ticket creation pipeline, API configuration, recovery actions |
 
---
 
## Skills Demonstrated
 
- Linux server administration (Ubuntu Server)
- Zabbix deployment and production configuration
- Agent-based monitoring across Windows and Linux platforms
- Windows service and Active Directory health monitoring
- Threshold design and alert severity tiering
- REST API integration between monitoring and ITSM platforms
- NOC-style dashboard design and operational workflow
- Closed-loop alert-to-ticket automation
---
 
*Part of the [MedNet Enterprise Lab](../README.md) — Enterprise Healthcare IT Infrastructure & Security Operations Home Lab*
