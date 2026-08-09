# VLAN and Inter-VLAN Routing

## Overview

This document covers the migration of every MedNet host off the original flat `192.168.56.0/24` network onto its designated VLAN, the architectural decision governing how the environment is administered post-segmentation, and a detailed account of the most significant troubleshooting effort in the network infrastructure build: a routing defect that independently affected two unrelated multi-homed hosts and took the better part of a session to isolate.

---

## VLAN Architecture

| VLAN | Subnet | Purpose | Hosts |
|---|---|---|---|
| 10, Servers | 10.10.10.0/24 | Domain controller, file server | `dc01` (`WIN-1UKKKVRDHPB`), `mednet-fs01` |
| 20, Clinical | 10.10.20.0/24 | Clinical workstations | WS-CLIN-01 |
| 30, Admin | 10.10.30.0/24 | Administrative workstations | WS-ADMIN-01 |
| 40, IT/Ops | 10.10.40.0/24 | IT staff, monitoring/SIEM | WS-IT-01, Zabbix (`mon01`), Wazuh (`siem01`) |
| 50, DMZ | 10.10.50.0/24 | Ticketing portal | osTicket (`itsm01`) |

---

## Migration Mechanism: Internal Networks, Not Trunking

Each VLAN corresponds to a dedicated VirtualBox Internal Network (`mednet-vlan10` through `mednet-vlan50`), with each pfSense OPT interface attached to one. VirtualBox Internal Networks provide genuine broadcast-domain isolation without requiring 802.1Q trunk configuration inside the guest OSes, which keeps the lab's complexity focused on firewall and routing logic rather than switch-port tagging.

![VirtualBox VM network settings showing an adapter attached to a mednet-vlan Internal Network](../screenshots/03-vlan-and-inter-vlan-routing_01.png)

### A Critical Distinction: Internal Network vs. Host-only Adapter

The original flat network (`192.168.56.0/24`) was a VirtualBox **Host-only Adapter**, a network type that bridges the physical host machine directly to every VM attached to it. Internal Networks are structurally different: they are VM-to-VM only, by design, with the host machine deliberately excluded.

This distinction surfaced directly during the Wazuh dashboard migration. After moving `siem01` from the host-only network to `mednet-vlan40`, a `curl` request to the dashboard's new address from the Arch host hung indefinitely rather than failing cleanly. The interface, firewall, and dashboard service were all confirmed healthy; the actual cause was that the host machine no longer had any path onto that segment at all. The address hadn't changed ownership, the underlying network fabric had.

This meant every subsequent VLAN needed a deliberate answer to "how does an administrator actually reach these hosts," rather than assuming direct host access would simply continue to work.

---

## The Jump-Host Decision

Three options were considered for post-segmentation administration:

1. **Jump host on the network**: administer from a VM already inside the target VLAN.
2. **Dedicated management path**: add a Host-only Adapter as an additional pfSense interface purely for host-to-lab administrative access.
3. **WAN port-forwarding**: use VirtualBox NAT rules for occasional access.

Option 1 was chosen as the most representative of real-world practice: administrators in a production network typically reach restricted segments through a bastion or jump host, not a flat path from their own workstation. WS-IT-01 (Ubuntu Desktop, IT/Ops workstation) serves this role naturally, since VLAN 40 already represents the IT/Ops network it belongs to. This is not a workaround; it reflects the intended access model for the environment.

A useful downstream effect of this choice, confirmed later in the DMZ work: it also enforces a real access boundary, not just a topological one. A nurse's workstation on VLAN 20 has no path to the Wazuh or Zabbix dashboards, and no need for one.

![WS-IT-01 network configuration confirming its role as the mednet-vlan40 jump host](../screenshots/03-vlan-and-inter-vlan-routing_02.png)

---

## Host Migration Summary

| Host | Prior Network | Current Network | Address |
|---|---|---|---|
| Domain Controller | Host-only (192.168.56.10) | `mednet-vlan10` | 10.10.10.10 |
| File Server (`mednet-fs01`) | Host-only (192.168.56.20) | `mednet-vlan10` | 10.10.10.20 |
| WS-CLIN-01 | Host-only | `mednet-vlan20` | 10.10.20.100 (DHCP) |
| WS-ADMIN-01 | Host-only | `mednet-vlan30` | 10.10.30.100 (DHCP) |
| WS-IT-01 | Host-only | `mednet-vlan40` | 10.10.40.10 |
| Zabbix (`mon01`) | Host-only (192.168.56.30) | `mednet-vlan40` | 10.10.40.30 |
| Wazuh (`siem01`) | Host-only (192.168.56.50) | `mednet-vlan40` | 10.10.40.50 |
| osTicket (`itsm01`) | Host-only (192.168.56.120) | `mednet-vlan50` | 10.10.50.120 |

A recurring finding across nearly every migration: the VM's actual registered name in VirtualBox frequently did not match its role-based name in documentation or inventory (for example, WS-IT-01 is registered as "Ubuntu Host"). `VBoxManage list vms` was used before every migration to confirm the real name rather than assume it, after an early attempt against the assumed name `WS-IT-01` failed outright.

| | |
|---|---|
| ![Domain controller network configuration showing the migrated 10.10.10.10 address](../screenshots/03-vlan-and-inter-vlan-routing_03.png) | ![WS-CLIN-01 network configuration showing a DHCP-leased 10.10.20.x address](../screenshots/03-vlan-and-inter-vlan-routing_04.png) |

---

## Verifying Inter-VLAN Routing

Ping alone confirms reachability but not the path taken. TTL provides a simple, free proof that traffic actually crossed a router hop rather than being locally bridged: a same-subnet reply from a Linux host returns at `ttl=64`, while a reply that has passed through one router hop returns at `ttl=63` (Linux-to-Linux) or, when the responding host is Windows (default starting TTL 128), at `ttl=127`.

A ping from WS-IT-01 (VLAN 40) to the domain controller (VLAN 10, `10.10.10.10`) returned `ttl=127`, one less than Windows' default starting value of 128, confirming the packet genuinely traversed pfSense rather than being locally bridged within a single broadcast domain.

![Ping from WS-IT-01 to the domain controller showing ttl=127](../screenshots/03-vlan-and-inter-vlan-routing_05.png)

---

## Troubleshooting: The Routing Metric Collision

The most significant issue encountered in this phase was not a firewall misconfiguration at all, despite initially presenting as one, and despite over an hour of investigation strongly suggesting otherwise.

### Symptom

With osTicket migrated to VLAN 50 and a single, correctly scoped pfSense rule permitting `osTicket &rarr; DC:636` (LDAPS), a connection test from osTicket to the DC returned an immediate, consistent **"Connection refused."** Not a timeout; a fast, active rejection, repeated identically across multiple attempts.

![osTicket LDAPS connection test returning an immediate Connection refused](../screenshots/03-vlan-and-inter-vlan-routing_06.png)

### The Investigation

An immediate, active refusal is a meaningful signal in pfSense: per its own documentation, only a Reject action (or an application on the receiving end actively closing the connection) produces a TCP RST. A silent Block, and pfSense's implicit default-deny, both drop packets without a response, which would present as a timeout instead. This pointed the investigation toward "something is actively saying no" rather than "something is silently dropping," and every check made under that assumption came back clean:

- **DC port state:** `Get-NetTCPConnection -LocalPort 636` confirmed LDAPS listening on `0.0.0.0`, all interfaces, both address families.
- **Windows Firewall:** the inbound LDAPS rule was enabled, scoped to `Any` remote address, and applied to the `Any` profile.
- **Network Location Awareness:** both DC interfaces reported `DomainAuthenticated`, ruling out a firewall-profile misclassification silently disabling the rule.
- **pfSense OPT4 rule set:** confirmed to contain exactly the one intended LDAPS rule, no stray Reject rules, no Floating rules present.
- **pfSense Firewall log:** contained no entries matching the traffic in question at all, though this was later understood to be inconclusive rather than exculpatory, since a passing rule without logging explicitly enabled produces no log entry regardless of outcome.

### The Actual Cause

The deciding check was **Diagnostics &rarr; States**, filtered across all interfaces rather than just OPT4. It returned zero results. Since pfSense only creates a state entry when it processes a packet at all, an empty state table meant the traffic had never reached the firewall in the first place; every prior check on pfSense's rules and the DC's own services had been correct, because the problem was never on that side of the connection.

![Diagnostics > States filtered across all interfaces returning zero results for the LDAPS traffic](../screenshots/03-vlan-and-inter-vlan-routing_07.png)

Both the domain controller and the osTicket host are multi-homed: a NAT-attached adapter for internet access, and a lab-facing adapter for VLAN traffic. On both machines, independently, the OS's default route selection favored the NAT adapter over the lab-facing one for destinations outside the local subnet, despite the lab-facing adapter having an explicitly configured gateway:

- **On the DC:** both interfaces reported an identical interface metric (25) initially. Setting a distinct `InterfaceMetric` did not resolve it, because Windows computes effective route cost as `InterfaceMetric + RouteMetric` combined, and the LAN adapter's auto-calculated `RouteMetric` (256, typical for a statically configured gateway) still outweighed the NAT adapter's auto-calculated value (0, typical for a DHCP-obtained route) even after the interface metric was corrected. The fix required setting `RouteMetric` explicitly on the default routes themselves:
  ```powershell
  Set-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix 0.0.0.0/0 -RouteMetric 1
  Set-NetRoute -InterfaceAlias "Ethernet 2" -DestinationPrefix 0.0.0.0/0 -RouteMetric 500
  ```
- **On osTicket:** the equivalent Linux issue, a one-point metric difference (NAT at 100, lab interface at 101) left the NAT route winning by a hair. Fixed via NetworkManager:
  ```bash
  nmcli con mod "Host-only" ipv4.route-metric 50
  ```

In both cases, packets addressed to a valid destination on a valid, correctly firewalled network were instead sent out an adapter with no route to that destination at all. VirtualBox's NAT engine has no way to deliver a packet addressed to a private lab subnet it doesn't own, which produced the fast, active-looking refusal that made this appear, for over an hour, like a firewall or application-layer problem.

| | |
|---|---|
| ![Set-NetRoute output correcting the DC's default route metrics](../screenshots/03-vlan-and-inter-vlan-routing_08.png) | ![nmcli output correcting the osTicket host's route metric](../screenshots/03-vlan-and-inter-vlan-routing_09.png) |

### Lesson

When a service is confirmed healthy end to end (listening, firewalled correctly, application-layer configuration correct) but still unreachable, verify the *live* routing table on both endpoints before continuing to audit the firewall or the application. A multi-homed host with more than one plausible default route is a routing problem until proven otherwise, and pfSense's state table is the fastest way to determine which side of the connection actually owns the problem: an empty state table means look at the origin host, not the firewall.

---

## Related Documents

| Document | Description |
|---|---|
| [README.md](README.md) | Network Infrastructure module overview and documentation index |
| [01-network-segmentation-design.md](01-network-segmentation-design.md) | VLAN architecture rationale and subnet design |
| [02-firewall-and-routing-configuration.md](02-firewall-and-routing-configuration.md) | Interface and firewall rule configuration |
| [04-security-hardening.md](04-security-hardening.md) | Deferred hardening items and rationale |

---

*Part of the [MedNet Enterprise Lab](../README.md), an Enterprise Healthcare IT Infrastructure & Security Operations home lab.*
