# Security Hardening

## Status

**Not yet implemented.** This document is a scaffold for now — it lays out the planned scope so the doc is ready to fill in with real configuration steps, verification, and any troubleshooting as each part gets done. Following the same practice as Modules 01 and 05, this file is meant to reflect hardening actually applied, not a plan checked off in advance.

## Overview

The `06-MedNet-SIEM` README currently lists two open items from initial deployment: the dashboard is still on Wazuh's default self-signed certificate rather than one issued by `MedNet-RootCA`, and dashboard authentication isn't yet tied to AD. This document also owns a third, smaller item: the `firewalld` port discrepancy flagged in `01-architecture-and-deployment.md`, where ports 1514–1515 were described as opened at initial deployment but weren't actually present in the live internal zone during agent rollout. Rough estimate is a weekend across the four parts below.

## Part 1: TLS Certificate Replacement (Wazuh Dashboard)

**Planned scope**
- Replace the dashboard's default self-signed certificate with one issued by `MedNet-RootCA`, the same CA already trusted by the other Linux service VMs in this environment.
- Certificate must include the dashboard's FQDN in the SAN, not just the CN — the same gotcha hit with the pfSense GUI cert; CN-only certs get flagged "Not Secure" by modern browsers.

**To document once complete**
- FQDN used, and whether a new DNS A-record was needed (same pattern as `dc01`, which was only a conceptual name until an alias record existed)
- Verification steps
- Any issues hit along the way

## Part 2: LDAPS Authentication (svc_wazuh)

**Planned scope**
- Bind dashboard authentication to AD over LDAPS using the `svc_wazuh` service account, already provisioned — same pattern used for `svc_osticket` in Module 03.

**To document**
- LDAPS configuration details
- Port/cert verification via `openssl s_client` — raw TCP checks like `cat` produced a false negative on Wazuh's TLS-only port 1515 during agent rollout, so the same tool applies here
- Any authentication issues found

## Part 3: Agent Enrollment Password Hardening

**Planned scope**
- Move off whatever enrollment password/mechanism was used for the initial six-agent rollout, toward something less reusable per host.

**To document**
- Approach taken
- Whether existing enrolled agents (`dc01`, `mednet-fs01`, `itsm01`, `WS-IT-01`, `WS-CLIN-01`, `WS-ADMIN-01`) needed re-enrollment or could be updated in place

## Part 4: firewalld Port Persistence (1514/1515)

**Planned scope**
- Resolve the discrepancy in `01-architecture-and-deployment.md`: ports 1514–1515 were documented as opened at initial deployment but weren't present in the live internal zone during the Module 06 session. Confirm current state, correct it, and make sure the ports are bound to the `internal` zone on the lab-facing adapter (not `public`/NAT — the same zone-to-adapter mismatch has silently broken rules elsewhere in this project) and set to persist across reloads.

**To document**
- Root cause of the original discrepancy
- Corrected configuration
- Verification method used

## Related Documents

| Document | Description |
|---|---|
| [README.md](README.md) | SIEM module overview and documentation index |
| [01-architecture-and-deployment.md](01-architecture-and-deployment.md) | Manager deployment and host firewall design; documents the certificate gap this doc closes |
| [02-log-source-and-agent-integration.md](02-log-source-and-agent-integration.md) | Per-host agent rollout; where the firewalld 1514/1515 discrepancy was first found |

---

*Part of the [MedNet Enterprise Lab](../README.md), an Enterprise Healthcare IT Infrastructure & Security Operations home lab.*
