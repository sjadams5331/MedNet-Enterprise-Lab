# Firewall and Routing Configuration

## Overview

This document covers the pfSense interface and firewall rule configuration underpinning MedNet's five-VLAN architecture. Where `01-network-segmentation-design.md` establishes the *why* behind the VLAN layout, this document covers the *how*: interface assignment, IP addressing, and the rule logic that governs what each segment can reach.

The guiding principle throughout is least privilege with visible intent: rather than a single broad allow rule per interface, each VLAN's outbound access is built from named, purposeful rules (a specific host, a specific port) stacked above a general block, so the ruleset itself documents the design decision rather than hiding it behind an opaque "Any/Any."

---

## Interface Assignment

| pfSense Interface | NIC | VLAN | Subnet | Internal Network Name |
|---|---|---|---|---|
| WAN | em0 (Adapter 1) | (uplink) | NAT, 10.0.2.0/24 | (NAT, not an Internal Network) |
| LAN | em1 (Adapter 2) | VLAN 10, Servers | 10.10.10.0/24 | `mednet-vlan10` |
| OPT1 | em2 (Adapter 3) | VLAN 20, Clinical | 10.10.20.0/24 | `mednet-vlan20` |
| OPT2 | em3 (Adapter 4) | VLAN 30, Admin | 10.10.30.0/24 | `mednet-vlan30` |
| OPT3 | em4 (Adapter 5) | VLAN 40, IT/Ops | 10.10.40.0/24 | `mednet-vlan40` |
| OPT4 | em5 (Adapter 6) | VLAN 50, DMZ | 10.10.50.0/24 | `mednet-vlan50` |

Each OPT interface maps to a dedicated VirtualBox Internal Network rather than a single trunked NIC carrying 802.1Q tags. This trades a small amount of realism (no VLAN tagging to practice against a physical switch) for architectural clarity in a lab context, and keeps each segment's broadcast domain genuinely isolated by VirtualBox itself rather than by firewall policy alone.

![pfSense Interfaces Assignments page showing WAN, LAN, and OPT1-4 all mapped and up](../screenshots/02-firewall-and-routing-configuration_01.png)

---

## Base Interface Configuration

Every OPT interface follows the same base pattern:

- **IPv4 Configuration Type:** Static IPv4, `10.10.X0.1/24`
- **IPv4 Upstream Gateway:** None (these are internal LAN-type segments, not WAN-facing; setting a gateway here would cause pfSense to treat the interface as a WAN type)
- **Block private networks / Block bogon networks:** both left unchecked. These options exist to protect WAN-facing interfaces from spoofed RFC1918 or unallocated source addresses. Every address on this network *is* RFC1918 by design, so enabling either would block the interface's own legitimate traffic.

---

## WAN

WAN is NAT-attached rather than host-only, receiving `10.0.2.15/24` (address may vary) from VirtualBox's built-in NAT DHCP. This is the sole path to the internet for the entire lab; every VLAN reaches external resources by routing through pfSense and out this interface.

![WAN interface status showing the NAT-assigned address and link state](../screenshots/02-firewall-and-routing-configuration_02.png)

---

## LAN (Servers, VLAN 10)

LAN carries the two pieces of shared infrastructure every other VLAN depends on: the domain controller and the file server. Two rules exist here:

1. **Anti-Lockout Rule** (pfSense-managed, not user-editable): permits GUI access (80/443) from the LAN subnet at all times, regardless of any other rule changes. This exists specifically to prevent a rule misconfiguration from locking out the only interface with console-adjacent access to the firewall.
2. **Default Pass (IPv4 \*/\*/\*)**: pfSense's stock allow-all rule, left in place deliberately for now. LAN's role is different from the client VLANs: most traffic to it is *inbound*, initiated by other segments (a DNS query, a Kerberos exchange, an SMB connection), and pfSense's state table handles the corresponding return traffic automatically without needing an outbound rule to permit it. Narrowing LAN's own outbound reach (limiting the DC and file server to only Windows Update, NTP, and CRL/OCSP endpoints rather than unrestricted internet) is scoped as future work; see `04-security-hardening.md`.

---

## The RFC1918_Private Alias

To build a "reach specific internal resources, otherwise internet-only" posture for the client VLANs without hand-writing a block rule for every other VLAN pair, a single alias covers all private address space:

| Name | Type | Networks |
|---|---|---|
| `RFC1918_Private` | Network(s) | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` |

A single rule blocking this alias, placed after the specific allow rules and before a catch-all internet pass, closes off lateral movement to every other internal segment in one line rather than one rule per destination.

![Firewall Aliases page showing the RFC1918_Private alias definition](../screenshots/02-firewall-and-routing-configuration_03.png)

---

## OPT1 (Clinical) and OPT2 (Admin)

Both client VLANs share an identical four-rule structure, since their infrastructure dependencies and threat model are the same. Order matters here: pfSense evaluates top to bottom on a first-match basis, so specific allows must sit above the broad block, and the broad block must sit above the final catch-all.

| # | Action | Destination | Port | Purpose |
|---|---|---|---|---|
| 1 | Pass | `10.10.10.10` (DC) | Any | Domain auth, DNS, GPO, NTP |
| 2 | Pass | `10.10.10.20` (File server) | TCP/445 | Department share access |
| 3 | Pass | `10.10.50.120` (osTicket) | TCP/443 | Ticket submission |
| 4 | Block | `RFC1918_Private` alias | Any | Deny lateral movement to every other internal segment, including each other |
| 5 | Pass | Any | Any | Everything surviving rule 4 is, by definition, non-private, i.e. internet-bound |

**On rule 1's "Any" protocol:** Active Directory's dependent services (Kerberos, LDAP, SMB for GPO pull, NTP, and assorted dynamic RPC ports) collectively span a wide and partly dynamic port range. Rather than enumerate each one, access is scoped by *destination host* instead: the DC is the single trusted target, so the security value comes from restricting where this traffic can go, not from micromanaging which port it uses to get there.

**On rule 4 being explicit rather than implicit:** once rules 1 through 3 are in place, pfSense's own implicit deny-all would eventually catch inter-VLAN traffic on its own, since no other rule permits it. An explicit Block rule is used instead so the isolation decision is a visible, intentional line in the ruleset, not an accidental side effect of what's absent.

**Clinical and Admin cannot reach each other.** This is deliberate. Keeping clinical systems isolated from administrative and business systems is a meaningful, defensible segmentation boundary in a healthcare-modeled network, not just a default. Prior to this rule set, both VLANs carried a single unrestricted pass-any rule, which meant Clinical and Admin were logically separate but functionally flat with each other; rule 4 closes that gap.

| | |
|---|---|
| ![OPT1 (Clinical) rule set showing the four rules in evaluation order](../screenshots/02-firewall-and-routing-configuration_04.png) | ![OPT2 (Admin) rule set, identical structure to OPT1](../screenshots/02-firewall-and-routing-configuration_05.png) |

---

## OPT3 (IT/Ops)

OPT3 currently retains a single, unrestricted **Pass / Any / Any / Any** rule. This is a deliberate exception, not an oversight: VLAN 40 is the management and monitoring tier. It hosts the jump host (see `03-vlan-and-inter-vlan-routing.md`) and will host Zabbix and Wazuh agent traffic to endpoints across every other VLAN once agent deployment begins (Modules 04 and 06). Applying the same RFC1918 block used on OPT1/OPT2 here would directly undermine the reason this VLAN exists.

Scoping OPT3 down to its actual required ports (Zabbix agent 10050/10051, Wazuh 1514/1515, plus DC and file-server access) is planned once real agent traffic exists to scope against, rather than guessing at ports prematurely. This is tracked as deferred work in `04-security-hardening.md`.

![OPT3 rule set showing the single Pass/Any/Any/Any rule](../screenshots/02-firewall-and-routing-configuration_06.png)

---

## OPT4 (DMZ)

OPT4 hosts osTicket, the one service in the environment reachable by every department. It is treated with the opposite posture from OPT1 through OPT3: instead of broad outbound access narrowed by a block, it gets exactly one narrow allow and nothing else.

| Action | Destination | Port | Purpose |
|---|---|---|---|
| Pass | `10.10.10.10` (DC) | TCP/636 (LDAPS) | Staff authentication only |

No default internet pass, no RFC1918 block needed (there is no broad rule left to be caught by one). pfSense's implicit deny-all handles everything else outbound from this interface without any further rules.

**Why osTicket sits in its own segment at all:** two reasons compound. First, osTicket is not domain-joined; it trusts the DC only through application-level LDAPS authentication, not the Kerberos trust relationship the domain-joined Windows hosts and WS-IT-01 share. That already makes it a lower-trust host by design. Second, a ticketing portal is inherently exposed to every user in every department, which makes it the most likely thing in the environment to eventually have a vulnerability found and exploited. Isolating it means that if it is ever compromised, whoever is on it lands in a dead end: no path to Clinical's data, no path to Admin's data, no path to the IT/Ops monitoring stack. The design principle isn't "isolate it from everything," it's "let people in, don't let it out": broadly reachable inbound (via the OPT1/2/3 rule 3 entries above), tightly restricted outbound.

![OPT4 rule set showing the single LDAPS allow to the domain controller](../screenshots/02-firewall-and-routing-configuration_07.png)

---

## Verifying Rule Behavior: pfSense State Table vs. Firewall Logs

Two different pfSense tools answer two different questions, and it is worth being precise about which to use when.

**Status &rarr; System Logs &rarr; General** records *configuration changes*, such as a rule being saved, edited, or reordered. Useful for an audit trail, not for understanding what traffic actually did.

**Status &rarr; System Logs &rarr; Firewall** records individual pass/block *decisions*, but only for rules that have logging explicitly enabled, and only within a rotating buffer (500 entries by default). A rule passing traffic silently and correctly will not appear here unless logging was turned on for it.

**Diagnostics &rarr; States** is the live connection table and the most reliable tool for answering "did this specific packet reach pfSense at all." An empty state table for a given destination and port, filtered across all interfaces, means the traffic never arrived at the firewall in the first place, which points the investigation toward the originating host's own routing rather than anything in the ruleset. This distinction became the deciding factor in a significant troubleshooting effort documented in `03-vlan-and-inter-vlan-routing.md`.

![Diagnostics > States table showing live connection entries used to verify rule behavior](../screenshots/02-firewall-and-routing-configuration_08.png)

---

## Related Documents

| Document | Description |
|---|---|
| [README.md](README.md) | Network Infrastructure module overview and documentation index |
| [01-network-segmentation-design.md](01-network-segmentation-design.md) | VLAN architecture rationale and subnet design |
| [03-vlan-and-inter-vlan-routing.md](03-vlan-and-inter-vlan-routing.md) | Host migration, inter-VLAN routing verification, and troubleshooting |
| [04-security-hardening.md](04-security-hardening.md) | Deferred hardening items and rationale |

---

*Part of the [MedNet Enterprise Lab](../README.md), an Enterprise Healthcare IT Infrastructure & Security Operations home lab.*
