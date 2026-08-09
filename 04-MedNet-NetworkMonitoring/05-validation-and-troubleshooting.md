# Validation and Troubleshooting

## Validation Approach

Validation methodology differed by check type, and using the wrong tool for a given host type produced misleading results more than once during this rollout.

**For passive-check hosts**, `zabbix_get` run directly from `mon01` was the primary validation tool:

```bash
zabbix_get -s <host-ip> -p 10050 -k agent.ping
```

A response of `1` confirms the entire chain, firewall, agent config, and hostname match, in a single command, independent of whatever the Zabbix frontend currently shows.

**For active-check hosts**, `zabbix_get` does not apply; it queries an agent the way a passive check would, which is the opposite of how these two hosts communicate. Validation instead relied on:

- The agent's own log (`zabbix_agent2.log`), specifically looking for `active check configuration update ... succeeded` vs. a specific failure like `rejected, allowed hosts` or `i/o timeout`
- Zabbix's **Latest Data** view, checking for populated values on real metric items (CPU, memory), not the host's own "Active agent availability" meta-item, which is calculated server-side on its own schedule and can show a recent timestamp even when zero real data has arrived

**For firewall-layer diagnosis**, pfSense's Diagnostics → States table, filtered by port, was the fastest way to determine whether a connectivity problem was network-side or host-side before spending time on either. An empty States table for the expected traffic meant the packet never reached pfSense in a state-creating way (either no matching rule, or a block rule intercepting it); a populated but one-sided state (e.g., `SYN_SENT:CLOSED` with no response) meant the packet arrived but got no answer, pointing at the destination host itself.

## Real Incidents

This section documents actual problems encountered during this module's rollout, in the order they occurred, with root cause and resolution. The goal, consistent with this project's broader documentation style, is to show structured troubleshooting and honest accounting of what actually happened, not a sanitized version where everything worked the first time.

### 1. Repurposed VM running an EOL Zabbix version

**Symptom:** N/A, discovered proactively before onboarding any hosts.
**Cause:** The Zabbix server VM was reused from an earlier, unfinished lab still running Zabbix 6.4, which reached end of support June 30, 2024.
**Resolution:** Upgraded in place to 7.0.29 LTS after a VM snapshot and full DB/config backup: repo pointer swap, package upgrade, automatic schema migration on first start. Verified via Help → About in the frontend post-upgrade.

### 2. Self-monitoring host stuck in a heartbeat failure loop

**Symptom:** `zabbix_server.log` repeatedly logging `cannot process heartbeat from host "Zabbix server": host not found` immediately after the 6.4→7.0 upgrade.
**Cause:** Two separate issues stacked together, classic `zabbix-agent` and `zabbix-agent2` were both installed and fighting over port 10050 (agent2 crash-looping with `address already in use`), and the surviving agent's `Hostname` value (`Zabbix server`, later `ZabbixLab`) didn't match the host's configured name in the frontend (`Zabbix`, later renamed to `mon01`).
**Resolution:** Disabled the classic agent, standardized on Agent 2, and aligned the `Hostname=` config value with the frontend's Host name field. Renamed the self-monitoring host to `mon01` to match the environment's naming convention, while leaving the underlying OS hostname (`ZabbixLab`) untouched.

### 3. `dc01` agent never actually installed

**Symptom:** A pre-existing host entry for the DC showed zero data ever collected (grey/inactive availability badge), even before any new configuration work began.
**Cause:** The host object existed in Zabbix from the prior, unfinished lab attempt, but the agent itself had never been installed on the DC. Windows agent config file search failures confirmed this directly.
**Resolution:** Fresh Agent 2 install, correct `Server`/`ServerActive`/`Hostname` values, Windows Firewall exception, validated with `zabbix_get`.

### 4. `mednet-fs01` silently dropping inbound connections (UFW)

**Symptom:** Passive check timed out after the correct pfSense rule was confirmed in place.
**Cause:** UFW was active on this host, contrary to an initial assumption that a minimal Debian install would have no local firewall. UFW's default deny policy drops silently, no rejection response, which is indistinguishable from "nothing listening" without a packet capture.
**Resolution:** `tcpdump` on the host confirmed SYNs arriving with no response; `ss -tlnp` confirmed the agent genuinely was bound and listening, ruling out a dead service. Added an explicit UFW allow rule for `mon01`'s IP on TCP/10050.
**Lesson:** Never assume a host has no local firewall; check the live ruleset directly (`ufw status`, `nft list ruleset`), on every host, before troubleshooting anything further upstream.

### 5. Rule scoped to a single host instead of a subnet

**Symptom:** `dc01`'s passive check worked; `mednet-fs01`'s, on the same VLAN, timed out with an empty pfSense States entry.
**Cause:** The original rule's Destination was a literal host IP (`10.10.10.10`) rather than the subnet.
**Resolution:** Changed Destination to the "LAN subnets" alias. Documented in full in `03-network-and-firewall-configuration.md`, since this recurred in a different form later (see #8).

### 6. `itsm01` (DMZ) had no general internet access

**Symptom:** `apt update` failing to resolve any hostnames, then failing entirely with a NetworkManager connection-activation error.
**Cause:** The DMZ (OPT4) has no default outbound rule by design. Getting package installation working required scoped rules for HTTP/HTTPS/DNS specifically, plus a genuinely separate NIC-activation failure that needed a terminal-level `nmcli` fix rather than the GUI toggle.
**Resolution:** See `03-network-and-firewall-configuration.md` for the specific rules added. `ping` remains blocked by design; `apt update` succeeding is the correct validation signal for this host, not ICMP reachability.

### 7. Wrong Windows agent binary installed (`WS-ADMIN-01`)

**Symptom:** Agent service running, but `Get-Service` showed `Zabbix Agent`, not `Zabbix Agent 2`, and the agent log showed the connection being actively rejected (`allowed hosts: "127.0.0.1"`).
**Cause:** Assumed incorrectly that the Windows MSI installer had a component-selection screen for choosing Agent vs. Agent 2. It does not, they are two entirely separate MSI files, distinguished only by the filename (`zabbix_agent-...` vs. `zabbix_agent2-...`). The download page's own selector UI does not make this obvious.
**Resolution:** Uninstalled the classic agent, downloaded the correct `zabbix_agent2-...` MSI directly by URL, reinstalled with the correct component this time confirmed via `Get-Service`.

### 8. pfSense rule ordering against a default block

**Symptom:** After fixing #7 and correcting a config typo, active-check traffic from `WS-ADMIN-01` still timed out, with a completely empty States table despite an apparently-correct rule in place.
**Cause:** A default **Block → RFC1918_Private** rule sat above the new allow rule in OPT2's rule list. pfSense stops at the first match; the block fired first.
**Resolution:** Manually reordered the allow rule above the block via drag-and-drop, Saved, Applied Changes.
**Lesson:** An empty States table with an apparently-correct rule in place means the *full, ordered* rule list needs review, not just the one rule that was edited.

### 9. Dual default gateways causing nondeterministic routing

**Symptom:** After fixing #8, traffic still timed out, and the pfSense States table remained completely empty, meaning the packet was never reaching the firewall at all this time.
**Cause:** `WS-ADMIN-01` has two active NICs (VirtualBox NAT + host-only VLAN adapter) with tied default-route metrics. Windows was intermittently routing lab-destined traffic out the wrong adapter.
**Resolution:** Added an explicit static route for `10.10.0.0/16` via the correct VLAN gateway, removing the ambiguity rather than relying on Windows' tie-breaking behavior. `WS-CLIN-01` was later found to have the same latent ambiguity (tied metrics) despite appearing to work; the same explicit route was added there as a precaution rather than relying on it continuing to resolve favorably on every future boot.

### 10. Recurring pattern: editing config without restarting the service

This happened independently on `mon01`, `itsm01`, and `WS-ADMIN-01`: a config file was correctly edited (`Hostname`, `Server`, `ServerActive`), confirmed correct via direct file inspection, and yet the agent continued behaving as though the old values were still active. In every case, the cause was the same, the running process was still using whatever it loaded at last start, and had not been restarted since the edit. Each time, this cost real troubleshooting effort chasing an already-fixed problem before the stale-process explanation was considered.

**Standing lesson for future modules:** any config file edit to a running Zabbix agent should be followed immediately by a service restart as the very last step, before any validation is attempted, not as an afterthought once validation fails.

## Summary

Every real issue in this module was diagnosable with a small, repeated set of tools: `zabbix_get` or the agent log for check-layer validation, pfSense's States table for network-layer validation, and direct inspection of live config/firewall state rather than trusting an earlier assumption about what "should" already be true. That States-table-first, verify-don't-assume approach is the most transferable takeaway from this module, more so than any single fix.
