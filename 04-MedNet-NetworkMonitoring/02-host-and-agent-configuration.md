# Host and Agent Configuration

## Agent Strategy: Two Distinct Models

This deployment uses two different Zabbix check models, chosen per host based on IP stability rather than applied uniformly:

**Passive checks** (`dc01`, `mednet-fs01`, `itsm01`, `WS-IT-01`, and `mon01` self-monitoring) are used for every host with a static IP. The server (`mon01`) initiates the connection to each agent on TCP 10050 and pulls data on its own schedule. This is the simpler, more common model and was used by default wherever a stable IP made it viable.

**Active checks** (`WS-CLIN-01`, `WS-ADMIN-01`) are used for the two clinical/admin workstations specifically because they receive DHCP addresses. Polling a host by an IP that can change on lease renewal is fragile; active checks invert the relationship, the agent dials out to the server on TCP 10051, using a configured hostname rather than depending on the server knowing the agent's current address. This adds real complexity (see below and `05-validation-and-troubleshooting.md`) but avoids building monitoring on top of an assumption that will eventually break.

Every active-check host is linked to the **"active" variant** of its template (e.g., "Windows by Zabbix agent active") rather than the standard passive template. This matters more than it first appears: the standard template's items are passive-typed by default and require a polled interface. Linking the standard template to an active-only host produces a host that looks connected and healthy in the agent log while silently collecting zero data, since the server is still trying to poll it the old way. This was a real issue encountered during setup, not a hypothetical one.

## Agent vs. Agent 2

All hosts run **Zabbix Agent 2**, not the classic C agent, for consistency with `mon01` itself and to take advantage of its plugin architecture and more modern Go-based implementation. On Linux this is a straightforward package choice (`zabbix-agent2`). On Windows it is *not* a checkbox in a single installer, contrary to what the installer's own component list might suggest, Agent and Agent 2 are entirely separate MSI packages, distinguished only by whether `2` appears in the filename (`zabbix_agent-...` vs. `zabbix_agent2-...`). This distinction caused a real installation error during `WS-ADMIN-01`'s setup, documented in `05-validation-and-troubleshooting.md`, and is called out here so it isn't repeated on future hosts.

## Per-Host Configuration

| Host | Check type | Template | Zabbix Host name |
|---|---|---|---|
| `dc01` | Passive | Windows by Zabbix agent | `dc01` |
| `mednet-fs01` | Passive | Linux by Zabbix agent | `mednet-fs01` |
| `itsm01` | Passive | Linux by Zabbix agent | `itsm01` |
| `WS-IT-01` | Passive | Linux by Zabbix agent | `WS-IT-01` |
| `WS-CLIN-01` | Active | Windows by Zabbix agent **active** | `WS-CLIN-01` |
| `WS-ADMIN-01` | Active | Windows by Zabbix agent **active** | `WS-ADMIN-01` |
| `mon01` | Passive (self) | Linux by Zabbix agent, Zabbix server health | `mon01` |

For every host, the Zabbix **Host name** field must match the agent's own `Hostname=` configuration parameter exactly, character for character. This match matters for both check types but is stricter and more consequential for active checks, since the server correlates incoming active-check heartbeats to a configured host purely by this string; a mismatch produces log entries like `host not found` on the server side, with the host appearing to have no failures at all on the agent side.

### Alias Naming

Two hosts (`dc01`, `itsm01`) use a Zabbix-facing name that differs from the machine's actual OS-level hostname (`WIN-1UKKKVRDHPB` and `osticket01`, respectively). In both cases the real hostname was left untouched rather than renamed, since renaming a domain-joined machine or a host with existing service dependencies carries real risk for no real benefit. The Zabbix host name is treated as an alias layer on top of the real system identity, the same pattern used for DNS in other modules of this environment.

### DHCP Interfaces

`WS-CLIN-01` and `WS-ADMIN-01` do not have an Agent-type interface defined in Zabbix at all. Since every item on both hosts comes from the active-check template, no interface is actually polled, an interface field would sit unused and would eventually go stale relative to whatever the current DHCP lease happens to be. Omitting it entirely is deliberate, not an oversight.

## Real Issues Encountered

A full account of what went wrong during agent deployment, and how each issue was diagnosed, lives in `05-validation-and-troubleshooting.md`. In summary, per host:

- **`mon01` (self):** stale `Hostname` value and a classic-agent/Agent-2 port conflict, both left over from the 6.4→7.0 upgrade.
- **`dc01`:** agent had never actually been installed despite an existing (non-functional) host entry from the prior lab.
- **`mednet-fs01`:** UFW was active and silently dropping inbound connections, undetected until a direct packet capture.
- **`itsm01`:** required narrowly-scoped outbound DMZ rules (HTTP/HTTPS/DNS only) before the agent package could even be installed.
- **`WS-ADMIN-01`:** wrong agent binary installed (classic instead of Agent 2), a config field typo, a pfSense rule-ordering conflict, and a dual-default-gateway routing ambiguity, four separate issues stacked on one host.
- **`WS-CLIN-01`:** appears to have avoided the routing issue by chance rather than by correct configuration; a deterministic static route was added afterward rather than relying on that.
