# Zabbix Integration
 
## Overview
 
One of the most operationally significant integrations in the MedNet environment is the automated ticket creation pipeline between Zabbix and osTicket. When Zabbix detects a threshold breach on any monitored host — high CPU, disk space exhaustion, a service going down — it automatically opens a ticket in osTicket routed to the NOC / Infrastructure department. This eliminates manual handoff between monitoring and the service desk and reflects how mature enterprise NOC environments operate.
 
---
 
## How It Works
 
Zabbix triggers an action when an alert condition is met. That action calls a script on `mon01.mednet.lab` that sends a POST request to the osTicket API, creating a ticket with relevant alert details pre-populated. When the alert resolves in Zabbix, a follow-up API call can update or close the corresponding ticket.
 
```
Zabbix threshold breach
        │
        ▼
Zabbix Action triggered
        │
        ▼
Script executes on mon01.mednet.lab
        │
        ▼
POST request to osTicket API (itsm01.mednet.lab)
        │
        ▼
Ticket created → routed to NOC / Infrastructure
        │
        ▼
Agent notified — SLA clock starts
```
 
---
 
## osTicket API Configuration
 
An API key was generated in osTicket scoped specifically to ticket creation from Zabbix. The key is associated with the IP address of `mon01.mednet.lab` to restrict where API requests can originate.
 
| Property | Value |
|---|---|
| API Key Scope | Ticket creation |
| Allowed IP | `mon01.mednet.lab` |
| Endpoint | `http://itsm01.mednet.lab/api/tickets.json` |
| Routed Department | NOC / Infrastructure |
| Help Topic | Infrastructure Alert (Zabbix) |
| Default Priority | High |
 
---
 
> 📸 **Screenshot:** osTicket API key configuration page showing the key and allowed IP
 
---
 
## Zabbix Action Configuration
 
A dedicated Zabbix action handles alert-to-ticket conversion. The action fires on any trigger of severity Warning or above across all monitored hosts.
 
### Action Conditions
 
| Condition | Value |
|---|---|
| Trigger severity | Greater than or equal to Warning |
| Hosts | All monitored hosts |
 
### Action Operations
 
| Operation | Details |
|---|---|
| Run remote command | Executes ticket creation script on `mon01.mednet.lab` |
| Ticket subject | `[Zabbix Alert] {HOST.NAME} — {TRIGGER.NAME}` |
| Ticket body | Host, trigger name, severity, current value, time of event |
 
### Recovery Operations
 
When a Zabbix trigger resolves, a recovery action sends a follow-up API call to osTicket appending a resolution note to the original ticket.
 
---
 
> 📸 **Screenshot:** Zabbix action configuration showing trigger conditions and remote command operation
 
> 📸 **Screenshot:** Zabbix trigger firing and the resulting ticket appearing in osTicket NOC queue
 
---
 
## Ticket Creation Script
 
The script runs on `mon01.mednet.lab` and is called by Zabbix as a remote command. It accepts Zabbix macros as arguments and formats them into the osTicket API payload.
 
```bash
#!/bin/bash
# zabbix-to-osticket.sh
# Called by Zabbix action on trigger event
 
OSTICKET_URL="http://itsm01.mednet.lab/api/tickets.json"
API_KEY="<osticket-api-key>"
 
HOST="$1"
TRIGGER="$2"
SEVERITY="$3"
VALUE="$4"
TIME="$5"
 
curl -s -X POST "$OSTICKET_URL" \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"Zabbix Monitoring\",
    \"email\": \"zabbix@mednet.lab\",
    \"subject\": \"[Zabbix Alert] $HOST — $TRIGGER\",
    \"message\": \"Host: $HOST\nTrigger: $TRIGGER\nSeverity: $SEVERITY\nValue: $VALUE\nTime: $TIME\",
    \"topicId\": \"<infrastructure-alert-topic-id>\",
    \"ip\": \"$(hostname -I | awk '{print $1}')\"
  }"
```
 
---
 
> 📸 **Screenshot:** Example auto-generated ticket in osTicket showing Zabbix alert details, department routing, and SLA status
 
---
 
## Why This Matters
 
Automated alert-to-ticket integration is a standard expectation in enterprise NOC environments. This integration demonstrates:
 
- Understanding of monitoring-to-ITSM workflows
- REST API usage for service integration
- Practical Zabbix action and macro configuration
- Closed-loop incident management from detection to resolution
In a real hospital environment, this type of integration ensures that infrastructure alerts are never silently acknowledged in a monitoring console — they become tracked, assigned, and SLA-bound work items.
 
---
 
*Part of the [MedNet Enterprise Lab](../../README.md)*
