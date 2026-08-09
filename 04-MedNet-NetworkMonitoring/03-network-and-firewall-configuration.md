# Network and Firewall Configuration

## Why This Exists as Its Own Document

The earlier, unfinished lab this Zabbix server was repurposed from ran on a single flat VirtualBox host-only network, no VLANs, no firewall boundaries between the monitoring server and its targets. MedNet's network is fully segmented behind pfSense, which means every monitoring path in this module had to be deliberately opened rather than assumed. This turned out to be the single largest source of real troubleshooting during this module's rollout, more than the Zabbix configuration itself, and is documented separately for that reason.

## Interface-to-VLAN Mapping

| pfSense Interface | VLAN | Subnet |
|---|---|---|
| LAN | 10 – Servers | 10.10.10.0/24 |
| OPT1 | 20 – Clinical | 10.10.20.0/24 |
| OPT2 | 30 – Admin | 10.10.30.0/24 |
| OPT3 | 40 – IT/Ops | 10.10.40.0/24 |
| OPT4 | 50 – DMZ | 10.10.50.0/24 |

`mon01` and `WS-IT-01` both live on VLAN 40/OPT3, no firewall rule was needed between them, they share a broadcast domain.

## Rules Added for Passive Checks (TCP 10050)

Passive checks require `mon01` (10.10.40.30) to reach each target agent inbound. Rules were added on **LAN** (for `dc01` and `mednet-fs01`) and **OPT4** (for `itsm01`), each permitting TCP/10050 from 10.10.40.30 to the target subnet.

## Rules Added for Active Checks (TCP 10051)

Active checks reverse the direction: the two DHCP workstations dial *out* to `mon01` on TCP 10051 (the Zabbix trapper port). Rules were added on **OPT1** (Clinical) and **OPT2** (Admin), each permitting TCP/10051 from the workstation's subnet to 10.10.40.30 specifically.

## Rules Added for DMZ Outbound (itsm01)

`itsm01` sits in the DMZ (OPT4), which does not have general outbound internet access by design. Package installation required three narrowly-scoped outbound rules rather than opening the DMZ broadly:

- TCP/443 (HTTPS) from 10.10.50.0/24 to any
- TCP/80 (HTTP) from 10.10.50.0/24 to any
- UDP/53 (DNS) from 10.10.50.0/24 to **This Firewall** specifically, with the host's DNS pointed at pfSense's own resolver (10.10.50.1) rather than a public resolver, so DNS traffic never needs to leave the DMZ at all

ICMP was deliberately left blocked. `ping` failing from `itsm01` to the outside world is expected and correct, `apt update` succeeding is the actual success condition for this host.

## Two Recurring Mistakes Worth Documenting

These weren't one-off typos, they're two distinct *categories* of pfSense mistake that recurred across multiple hosts, and are worth calling out explicitly so they don't repeat in future modules.

### 1. Scoping a rule to a single host instead of a subnet

The first passive-check rule (for `dc01`) was written with a literal destination address (`10.10.10.10`) rather than the subnet. It worked, correctly, for `dc01`, which meant the mistake wasn't visible until `mednet-fs01` (same VLAN, different IP) was added and its passive check timed out with no explanation. The fix in every case was the same: change the Destination (or Source, for the reverse-direction active-check rules) field's type from "Address or Alias" to the relevant interface's **subnets** alias (e.g., "LAN subnets," "OPT2 subnets"), rather than typing a CIDR by hand. pfSense's CIDR dropdown next to a manually-typed address has a known UI quirk where it can appear locked/greyed out after a paste; using the subnets alias sidesteps that entirely and is the preferred approach going forward.

**Diagnostic signature:** an empty entry in Diagnostics → States for the expected source/destination/port combination is the fastest way to confirm a rule isn't matching at all, before assuming the problem is on the target host.

### 2. Rule ordering against a default block

pfSense evaluates rules top-to-bottom per interface and stops at the first match. Both OPT1 and OPT2 have a default **Block → RFC1918_Private** rule (blocking general private-range traffic) positioned near the bottom of an otherwise permissive rule set. A newly-added allow rule placed *below* that block never takes effect, the traffic matches the block first and is silently dropped, producing an empty States table identical in appearance to the "wrong subnet" mistake above. The fix is a manual drag-reorder of the new rule to a position above the block, followed by Save and Apply Changes.

**Distinguishing the two:** both produce an empty States table. The rule-ordering issue is confirmed by reviewing the *full* rule list for the interface, in order, not just the one rule that was edited.

## Non-Firewall Networking Issue: Dual Default Gateways

Both Windows workstation VMs have two active NICs, a VirtualBox NAT adapter (for general internet access) and the host-only adapter connected to their VLAN. When both have a default route with the same metric, Windows breaks the tie using interface-level metrics that can vary based on boot order, which produced genuinely nondeterministic routing behavior; monitoring traffic destined for `mon01` would sometimes correctly exit via the VLAN adapter and sometimes incorrectly attempt to exit via the NAT adapter, timing out either way. This is not a pfSense issue and does not show up in pfSense's States table at all (the packet never reaches the firewall in the failure case).

**Fix:** rather than fight over which adapter owns the default route, an explicit static route was added on each workstation for the entire lab range:

```powershell
New-NetRoute -DestinationPrefix "10.10.0.0/16" -InterfaceAlias "<host-only adapter>" -NextHop "<VLAN gateway>"
```

This guarantees all `10.10.x.x` traffic uses the correct adapter regardless of default-route tie-breaking, while leaving general internet traffic on the NAT adapter untouched.

## Open Item

A stray, misconfigured rule was found on OPT1 during `WS-ADMIN-01` troubleshooting, its Source was incorrectly set to "OPT2 subnets" rather than "OPT1 subnets," meaning it matches almost no real traffic on that interface. This needs to be corrected or removed; as of this writing it has not yet been confirmed fixed.
