# Log Source and Agent Integration

## Rollout Approach

Agents were deployed one host at a time, in a deliberate order, validating each one on the dashboard before moving to the next, the same approach used for the Zabbix agent rollout in Module 04. The order was chosen for a mix of value and risk: `dc01` first as the highest-value log source, then the two remaining static-IP servers, then the DMZ host, then the two workstations that had previously caused the most trouble during the Zabbix rollout.

Every host uses the Wazuh dashboard's **Deploy new agent** wizard to generate the platform-specific install command, rather than a hand-written one, so the install string stays pinned to whatever manager version is actually running (4.14.4) rather than an assumed or remembered one.

## Interface-to-VLAN Mapping

This mapping was originally established during Module 04's Zabbix rollout and is restated here for reference, since consulting it directly (rather than re-deriving it) would have avoided one of the mistakes documented below.

| pfSense Interface | VLAN | Subnet |
|---|---|---|
| LAN | 10, Servers | 10.10.10.0/24 |
| OPT1 | 20, Clinical | 10.10.20.0/24 |
| OPT2 | 30, Admin | 10.10.30.0/24 |
| OPT3 | 40, IT/Ops | 10.10.40.0/24 |
| OPT4 | 50, DMZ | 10.10.50.0/24 |

`siem01` and `WS-IT-01` both sit on VLAN 40/OPT3, the same broadcast domain, meaning no firewall rule at all is required between them, matching the identical finding already documented for Zabbix in Module 04.

## Per-Host Rollout

### dc01

The agent installed and started cleanly on the first attempt, with `WAZUH_AGENT_NAME` explicitly set to `dc01` to preserve the same alias-over-real-hostname pattern used everywhere else in this environment (the machine's actual OS hostname, `WIN-1UKKKVRDHPB`, is left untouched). Enrollment itself failed, with the agent log repeating `ERROR: (1208): Unable to connect to enrollment service`.

`Test-NetConnection` against `siem01:1515` confirmed a full timeout rather than an active refusal, and pfSense's States table showed no entry at all for that traffic, meaning the packet was not reaching the firewall as expected. Reviewing the LAN interface's existing rule set found a rule already in place for this purpose, but with its Source field set to **OPT1 subnets** rather than **LAN subnets**, a plain wrong-selection mistake unrelated to the CIDR-typing bug documented in Module 04. Correcting the Source field and applying the change resolved it, and the agent enrolled on its next retry.

### mednet-fs01 (fs01)

Confirmed on VLAN 10 (`10.10.10.20`) via `ip addr`, already covered by the dc01 fix above, so no new pfSense rule was needed.

The Deploy Agent wizard's multi-line generated command did not paste cleanly into the VirtualBox console; `WAZUH_MANAGER` was dropped from the actual install invocation, though the `.deb` package still downloaded and installed successfully as two separate manual commands. The result looked like a normal, successful install right up until the service failed to start: `ERROR: (4112): Invalid server address found: 'MANAGER_IP'`. The installer's own templating had never received a real address to substitute, and left the literal placeholder text in `ossec.conf`.

Fixed with `sed`. The first attempt used a forward-slash delimiter with an escaped `\/` inside the replacement string, which after being retyped into the VM console produced a corrupted line containing a literal backslash and trailing period rather than a valid closing tag. The second attempt used a pipe (`|`) delimiter instead, which needs no escaping at all and resolved cleanly.

**Lesson:** a successful install and a "service started" message do not confirm environment variables were actually captured. Verifying the config file's real contents is worth doing before assuming a scripted or copy-pasted install worked as intended.

### itsm01

Confirmed on VLAN 50 (`10.10.50.120`, DMZ, OPT4) via `ip addr`. An inbound pass rule for TCP 1514-1515 was built on OPT4 using the OPT4 subnets alias from the start this time. The rule matched real traffic (its state counter incremented), and `ss -tlnp` on `siem01` confirmed `wazuh-authd` was genuinely listening on `0.0.0.0:1515`, yet the connection still would not complete.

The actual cause was on `siem01`'s own host firewall, not pfSense. `firewalld`'s `public` zone (bound to the NAT-facing `enp0s3`) had only `443/tcp` open; the `internal` zone (`enp0s8`, `siem01`'s real lab-facing interface, where `10.10.40.50` actually lives) did not have `1514-1515/tcp` open. Fixed with:

```bash
sudo firewall-cmd --permanent --zone=internal --add-port=1514-1515/tcp
sudo firewall-cmd --reload
```

This is worth flagging plainly against `01-architecture-and-deployment.md`, which describes the internal zone as having had these exact ports opened at initial deployment. Whether that description was inaccurate at the time, or the rule was lost at some point between deployment and this session, was not determined. The internal zone's actual port set should be periodically audited against intent rather than assumed static.

Even after both fixes were in place, a simple `cat < /dev/tcp/10.10.40.50/1515` connectivity test still reported closed/filtered, which briefly looked like a third problem. A `tcpdump` capture on `siem01` during a retest showed the full three-way TCP handshake completing successfully, and a follow-up `openssl s_client -connect 10.10.40.50:1515` test confirmed a complete TLSv1.3 handshake against Wazuh's self-signed certificate. The `cat`-based test was a false negative the whole time: port 1515 is TLS-only and expects the client to speak first, and `cat` simply sat on an open, idle socket until `timeout` killed it.

**Lesson:** a raw TCP read is not a valid check against a TLS-only service. `openssl s_client` is the correct tool for validating any Wazuh port going forward.

Separately, outbound internet access from `itsm01` (needed to download the agent package from `packages.wazuh.com`) hung on the initial connection, the same category of issue already documented in Module 04's DMZ outbound section, and for the identical reason: the existing OPT4 outbound rule for ports 80 and 443 had its Source typed as the bare address `10.10.50.0` with no CIDR suffix, which pfSense silently treats as a single, nonexistent host rather than the intended network. Fixed by switching both rules' Source to the OPT4 subnets alias.

With both the outbound package path and inbound enrollment path confirmed open, the agent installed and enrolled cleanly using the same single-line install pattern learned from `mednet-fs01`, avoiding a repeat of the paste-splitting issue.

### WS-IT-01

No firewall changes required, since it shares VLAN 40/OPT3 with `siem01` directly. Deployed and enrolled without incident.

### WS-CLIN-01

Confirmed on VLAN 20 (`10.10.20.100`) via `ipconfig`. A rule was initially built on **OPT2**, based on an assumption made mid-session rather than checking this environment's own existing documentation. Module 04's `03-network-and-firewall-configuration.md` had already established the correct interface mapping during the Zabbix rollout (OPT1 is Clinical, OPT2 is Admin); that mapping was not consulted before building this rule, and the resulting mistake cost real troubleshooting time that a two-minute check would have avoided.

The actual mismatch was confirmed by pulling each OPT interface's configured static address directly (`10.10.20.1/24` on OPT1, `10.10.30.1/24` on OPT2) and cross-referencing pfSense's States table, which showed all of `WS-CLIN-01`'s real traffic, including its attempted connection to Zabbix's trapper port, flowing through OPT1 and never OPT2.

The correct rule was built on OPT1 (Source: OPT1 subnets, Destination: `10.10.40.50`, ports 1514-1515). The original OPT2 rule was not deleted; see `WS-ADMIN-01` below for why. The agent enrolled successfully once traffic was routed through the correct interface.

### WS-ADMIN-01

OPT2 is genuinely Admin's interface, confirmed by `WS-ADMIN-01`'s real address (`10.10.30.100`) matching OPT2's configured subnet. The rule mistakenly built on OPT2 while troubleshooting `WS-CLIN-01` turned out to already be exactly the rule `WS-ADMIN-01` needed, and was left in place with an updated description rather than removed and rebuilt from scratch.

The agent deployed and enrolled without any additional issues, unlike the four separate problems this same host hit during the Zabbix rollout in Module 04 (wrong agent binary, a config typo, a rule-ordering conflict, and a dual-default-gateway routing ambiguity). Whether the clean result here is because that earlier routing fix is still holding, or simply because a correct firewall rule already happened to exist by the time this module reached it, was not isolated. Worth a quick confirmation that the static route added in Module 04 is still present, since it was not revisited during this session.

## Firewall Rules Added for Wazuh Agent Traffic (Summary)

| Interface | VLAN | Host(s) | Direction | Rule |
|---|---|---|---|---|
| LAN | 10, Servers | `dc01`, `mednet-fs01` | Inbound to `siem01` | TCP 1514-1515, Source: LAN subnets, Destination: 10.10.40.50 |
| OPT1 | 20, Clinical | `WS-CLIN-01` | Inbound to `siem01` | TCP 1514-1515, Source: OPT1 subnets, Destination: 10.10.40.50 |
| OPT2 | 30, Admin | `WS-ADMIN-01` | Inbound to `siem01` | TCP 1514-1515, Source: OPT2 subnets, Destination: 10.10.40.50 |
| OPT3 | 40, IT/Ops | `WS-IT-01` | n/a | None needed, same broadcast domain as `siem01` |
| OPT4 | 50, DMZ | `itsm01` | Inbound to `siem01` | TCP 1514-1515, Source: OPT4 subnets, Destination: 10.10.40.50 |
| OPT4 | 50, DMZ | `itsm01` | Outbound | TCP 80/443, Source: OPT4 subnets, Destination: any (corrected from a bare, non-CIDR address) |

Every rule above uses an interface subnet alias (for example, "LAN subnets," "OPT1 subnets") rather than a manually typed CIDR address, consistent with the standing recommendation already documented in Module 04.

## Host Firewall: siem01

`firewall-cmd --permanent --zone=internal --add-port=1514-1515/tcp`, followed by a reload, was required on `siem01` itself before any agent outside VLAN 40 could enroll. See the `itsm01` section above for the full diagnostic path and the open question about why this port range was not already present.

## Recurring Lessons

- **Check existing module documentation before rediscovering something through trial and error.** The Clinical/Admin interface mixup on `WS-CLIN-01` was already solved and documented in Module 04. Consulting it first would have saved real troubleshooting time.
- **An unscoped source address, a bare network address with no CIDR suffix, is silently treated by pfSense as a single, nonexistent host.** This exact pattern, already documented in Module 04, recurred on the `itsm01` outbound rule. Building rules with the interface's subnet alias instead of typing an address avoids this category of bug entirely and should be the default approach, not a fallback used after something breaks.
- **A raw TCP connection test is not a valid check against a TLS-only service.** Port 1515 expects a TLS handshake, not a passive read. Use `openssl s_client -connect <host>:<port>` for validating any Wazuh port going forward.
- **A successful install and a "started successfully" service message do not confirm environment variables were actually passed correctly.** The `mednet-fs01` placeholder-address bug looked identical to a working install right up until enrollment was attempted. Verifying a config file's real contents after any scripted or copy-pasted install is worth the extra step.

## Deferred

Screenshots for this document have not yet been captured or uploaded, consistent with the environment's overall screenshot plan of addressing Modules 02, 05, and 06 together once this module's core work was complete.

---

## Related Documents

| Document | Description |
|---|---|
| [README.md](README.md) | SIEM module overview and documentation index |
| [01-architecture-and-deployment.md](01-architecture-and-deployment.md) | Manager deployment, host firewall design, and initial DNS resolution |
| [../04-MedNet-NetworkMonitoring/03-network-and-firewall-configuration.md](../04-MedNet-NetworkMonitoring/03-network-and-firewall-configuration.md) | The original interface-to-VLAN mapping and firewall lessons this module drew on |

---

*Part of the [MedNet Enterprise Lab](../README.md), an Enterprise Healthcare IT Infrastructure & Security Operations home lab.*
