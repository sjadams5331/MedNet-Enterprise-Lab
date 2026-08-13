# Detection Rules and Alerting

## Status

**Not yet started.** Every agent is currently running only Wazuh's vendor-default ruleset — default File Integrity Monitoring plus a Security Configuration Assessment policy scoped to each host's OS (for example, `cis_win2025.yml` on `dc01`) — with no custom rules, decoders, or alert actions configured. This project's usual sequencing is to build and validate the actual detection content — including confirming alerts fire, not just that rules parse — before writing the doc for it, since this document is meant to record what was built and why, not a plan. This file exists as a scoping placeholder until that work happens.

## Overview

Once `04-security-hardening.md` is done, the next step is layering custom detection content on top of Wazuh's vendor defaults across the six log sources already rolled out in `02-log-source-and-agent-integration.md`: `dc01`, `mednet-fs01`, `itsm01`, `WS-IT-01`, `WS-CLIN-01`, and `WS-ADMIN-01`.

## Planned Coverage Areas

Rough outline based on what's already known about the environment — not yet built, and likely to shift once rule-writing actually starts:

- **Domain Controller (`dc01`)** — AD authentication anomalies: failed logons, account lockouts, unusual privileged group changes
- **File Server (`mednet-fs01`)** — unexpected file access or permission changes on Samba shares
- **ITSM (`itsm01`)** — DMZ-facing host, likely first priority for external-facing detection content
- **Workstations (`WS-CLIN-01`, `WS-ADMIN-01`, `WS-IT-01`)** — cross-VLAN traffic and lateral-movement alerting, tying back to the segmentation design in Module 05

## To Document Once Built

- Specific custom rules and decoders authored, and the reasoning behind them
- How alerts were validated — confirmed to actually fire, not just syntactically correct
- Any false positives found and tuned out
- Alerting actions/integrations configured (email, dashboard, etc.)

## Related Documents

| Document | Description |
|---|---|
| [README.md](README.md) | SIEM module overview and documentation index |
| [02-log-source-and-agent-integration.md](02-log-source-and-agent-integration.md) | Per-host agent rollout this detection content will build on |
| [04-security-hardening.md](04-security-hardening.md) | Planned hardening pass, recommended to complete first |

---

*Part of the [MedNet Enterprise Lab](../README.md), an Enterprise Healthcare IT Infrastructure & Security Operations home lab.*
