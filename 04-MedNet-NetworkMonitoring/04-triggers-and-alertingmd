# Triggers and Alerting

## Current State: Vendor Defaults Only

As of this writing, no custom triggers, actions, notification media, or escalation logic have been configured. Every trigger currently active on any host is whatever ships with the standard "Windows by Zabbix agent," "Linux by Zabbix agent," and their active-check counterparts, templates were deliberately left unmodified during host onboarding, consistent with this environment's broader preference for standard, supportable configuration over heavy customization.

This document exists to state that plainly rather than let the presence of a `04-triggers-and-alerting.md` file imply more was configured than actually was. Prior documentation for this project's earlier, unfinished lab attempt made claims about alerting philosophy, severity models, and validated alert lifecycles; none of that has been re-verified against this deployment, and none of it should be assumed to carry over.

## What the Default Templates Already Provide

The standard templates are not empty, out of the box they include reasonable baseline coverage:

- Host unreachable / agent not responding
- High CPU utilization (sustained)
- Low free disk space
- High memory utilization
- Zabbix agent restart / version change detection (self-monitoring host)

This baseline is a legitimate starting point, not a placeholder. For a lab whose current goal is proving the monitoring pipeline itself is reliable, host-down and resource-exhaustion coverage from the default templates is meaningful signal on its own.

## What's Missing for This to Be a Complete NOC-Style Deployment

- **No custom, role-specific triggers.** Nothing here reflects `dc01`'s role as a domain controller, `mednet-fs01`'s role as a file server, or `itsm01`'s role as a ticketing system specifically, every host is monitored identically as a generic Windows or Linux box. A small number of role-aware triggers (e.g., a specific disk-space threshold on the file server tied to its actual data volume, rather than a generic default) would meaningfully strengthen this module without requiring a large amount of new work.
- **No actions or notifications configured.** Triggers currently fire into the Problems view and nowhere else. There is no email, webhook, or other notification path defined, meaning an actual incident would only be noticed by someone actively watching the dashboard.
- **No escalation or severity tuning.** Default severities are whatever the vendor templates assign; none have been reviewed or adjusted for this environment's actual priorities (e.g., `dc01` going down is a more severe event here than a workstation's disk filling up, but nothing currently reflects that distinction).

## Recommendation

Given the scope of this module, full custom alerting is not a hard requirement to consider host-level monitoring "done", the default triggers already provide real availability and resource-exhaustion coverage. However, before treating this module as fully complete, at minimum:

1. Add one or two custom, host-specific triggers (not a large number, enough to demonstrate the capability and reasoning, not to build out a production alerting policy)
2. Configure at least one notification action so a triggered problem is actually surfaced somewhere outside the Zabbix UI
3. Revisit this document once that work is done, since its current framing (defaults only, explicitly scoped as incomplete) will no longer be accurate
