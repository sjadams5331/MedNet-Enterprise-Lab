# Helpdesk Configuration
 
## Overview
 
The osTicket configuration is designed around a realistic hospital service desk structure. Departments, roles, help topics, and SLAs reflect what you would find in a real healthcare IT environment — not a generic demo setup. This means clinical and administrative staff interact with the help desk differently, tickets are routed based on department and priority, and SLAs reflect the urgency appropriate for a healthcare setting.
 
---
 
## Departments
 
Tickets are routed to departments that mirror the operational structure of a hospital IT team.
 
| Department | Responsibility |
|---|---|
| Service Desk | General end user support, password resets, hardware issues |
| NOC / Infrastructure | Network, server, and infrastructure tickets — receives Zabbix alerts |
| Clinical Systems | Issues specific to clinical applications and devices |
| Security | Security incidents, access requests, policy violations |
 
---
 
> 📸 **Screenshot:** osTicket departments list showing all configured departments
 
---
 
## Roles & Permissions
 
Roles enforce separation of duties within the help desk. Each role maps to a realistic job function and carries only the permissions appropriate for that function.
 
| Role | Permissions |
|---|---|
| Tier 1 Support | Create, reply, close tickets within assigned department |
| Tier 2 Support | All Tier 1 permissions plus ticket reassignment, priority changes, SLA overrides |
| NOC / Infrastructure | View and manage infrastructure-related queues, limited access to user data |
| Administrator | Full system configuration and management |
 
Agent accounts are sourced from Active Directory and assigned roles within osTicket manually. The `MedNet-HelpDesk` AD security group is used to scope which AD accounts are permitted agent access.
 
---
 
> 📸 **Screenshot:** osTicket roles configuration page showing defined roles and permissions
 
---
 
## Help Topics
 
Help topics provide end users with a structured way to categorize their requests. They are aligned to a healthcare IT service catalog and drive automatic department routing.
 
| Help Topic | Routes To | Priority |
|---|---|---|
| Password Reset | Service Desk | Low |
| Hardware Issue | Service Desk | Normal |
| Software / Application Issue | Service Desk | Normal |
| Network Connectivity | NOC / Infrastructure | High |
| Clinical Application Issue | Clinical Systems | High |
| Security Incident | Security | Critical |
| New User Access Request | Service Desk | Normal |
| Infrastructure Alert (Zabbix) | NOC / Infrastructure | High |
 
---
 
> 📸 **Screenshot:** osTicket help topics list showing all configured topics and their routing
 
---
 
## Service Level Agreements (SLAs)
 
SLA tiers are defined by ticket priority and reflect the urgency appropriate for a healthcare environment where system availability directly impacts patient care.
 
| SLA Tier | Priority | Grace Period | Schedule |
|---|---|---|---|
| P1 — Critical | Critical | 1 hour | 24/7 |
| P2 — High | High | 4 hours | 24/7 |
| P3 — Normal | Normal | 8 hours | Business hours |
| P4 — Low | Low | 24 hours | Business hours |
 
SLAs are enforced automatically by osTicket based on help topic and department assignment. Overdue tickets are flagged in the agent view.
 
---
 
> 📸 **Screenshot:** osTicket SLA plans configuration showing grace periods and schedules
 
---
 
## Ticket Lifecycle
 
A standard ticket in the MedNet environment follows this workflow:
 
1. User submits ticket via web portal using Active Directory credentials
2. Help topic selection triggers automatic department routing
3. Agent receives and acknowledges ticket — SLA clock starts
4. Agent works ticket, adds internal notes as needed
5. User receives email updates at key lifecycle stages
6. Ticket resolved and closed — SLA compliance recorded
Escalation from Tier 1 to Tier 2 follows a defined process with priority and SLA adjustments logged on the ticket.
 
---
 
> 📸 **Screenshot:** Example ticket showing department routing, SLA status, and agent assignment
 
> 📸 **Screenshot:** User portal showing help topic selection and ticket submission form
 
---
 
*Part of the [MedNet Enterprise Lab](../../README.md)*
