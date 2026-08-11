# Architecture and Deployment

## Why Wazuh, and What It Is Not

Zabbix (Module 04) answers "is this host up and healthy." Wazuh answers a different question: "what happened on this host, and does it matter." It is the environment's SIEM and log aggregation layer, combining host-based intrusion detection, file integrity monitoring, security configuration assessment, and centralized log analysis into a single platform. The two tools are deliberately not merged or treated as redundant. Module 04's own documentation draws this same boundary from the Zabbix side: Zabbix is explicitly scoped to availability and resource monitoring, with SIEM responsibilities left to this module.

## Platform Choice

The manager runs on **Rocky Linux 9**, chosen for the same reason Rocky was picked for this role generally: it is RHEL-compatible, which is representative of what a real healthcare or enterprise environment would run for a security-sensitive service, and it gives direct, transferable exposure to `firewalld` and SELinux rather than Ubuntu's more permissive defaults.

Wazuh itself was installed using the **all-in-one installer**, which places the manager, indexer, and dashboard components on a single node. This is a reasonable and common choice at lab scale. A production SOC deployment would typically split these components across dedicated nodes for scalability and fault isolation, and that tradeoff is called out here rather than left implicit.

## Manager Host Details

| Property | Value |
|---|---|
| Hostname | `siem01.mednet.lab` |
| Operating System | Rocky Linux 9 |
| Wazuh Version | 4.14.4 |
| VLAN | 40, IT/Ops |
| Static IP | 10.10.40.50 |
| Agent Enrollment Port | 1515/tcp |
| Agent Communication Port | 1514/tcp |
| Dashboard Port | 443/tcp |

`siem01` shares VLAN 40 with the Zabbix server (`mon01`) and `WS-IT-01`, consistent with the environment's pattern of grouping IT and operations tooling on a single segment rather than scattering it across the network.

## Network Design

Like every other Linux host in this environment, `siem01` runs two network adapters: a VirtualBox NAT adapter (`enp0s3`) for outbound internet access, used only for package installation and updates, and a static host-only adapter (`enp0s8`) carrying its real lab identity at 10.10.40.50. This split keeps the manager's actual security posture scoped to the lab network, while still allowing it to pull updates without a dedicated proxy or repository mirror.

## Initial DNS Resolution Issues

Two DNS problems surfaced during initial deployment and were resolved before agent work could begin:

1. **Duplicate DNS registration on the domain controller.** `dc01` was registering DNS records for both its NAT adapter address and its real lab address in the `mednet.lab` zone. This produced round-robin DNS responses, and `siem01` would occasionally resolve the domain controller to an address it could not actually reach. This was resolved by disabling DNS registration on `dc01`'s NAT adapter (`Set-DnsClient -RegisterThisConnectionsAddress $False`) and removing the stale duplicate records from the forward lookup zone.
2. **IPv6 resolver priority overriding IPv4 DNS.** `siem01`'s own NAT adapter was receiving a DHCPv6-assigned nameserver that took priority over the intended IPv4 resolver pointed at the domain controller. This was resolved by disabling IPv6 entirely on both of `siem01`'s adapters, consistent with how IPv6 is handled elsewhere in this environment (it is not part of the lab's design and is disabled rather than partially configured).

## Host Firewall Design

`siem01` uses `firewalld` with a two-zone model that mirrors the same public/internal split used on the network side of the environment:

| Zone | Interface | Purpose | Inbound access |
|---|---|---|---|
| `public` | `enp0s3` (NAT) | Outbound package access only | None |
| `internal` | `enp0s8` (lab) | Wazuh application traffic | SSH, plus the Wazuh-specific ports below |

Default services not relevant to this role (`cockpit`, `dhcpv6-client`, `mdns`, `samba-client`) were removed from the internal zone rather than left at their defaults, following the same least-privilege approach used on the domain controller and file server.

**A note on scope for this document versus the next one:** the internal zone's exact set of open ports at any given point in time, and one gap found in it, is covered in `02-log-source-and-agent-integration.md` rather than restated here, since it came up directly during agent rollout troubleshooting rather than at initial deployment.

SELinux runs in enforcing mode with the targeted policy, with no AVC denials observed. `setroubleshoot-server` is installed to make any future denials easier to diagnose.

## Known Gaps

- **Single-node deployment.** Manager, indexer, and dashboard all run on one VM. Acceptable for lab scale, but not how a production SIEM would typically be architected.
- **Default self-signed certificate.** The dashboard and agent enrollment service are both still using Wazuh's default self-signed certificate rather than one issued by `MedNet-RootCA`. The internal CA is already trusted by the other Linux service VMs in this environment for exactly this kind of use case; replacing Wazuh's certificate with an internally-issued one is planned as part of `04-security-hardening.md` rather than done at initial deployment.

---

## Related Documents

| Document | Description |
|---|---|
| [README.md](README.md) | SIEM module overview and documentation index |
| [02-log-source-and-agent-integration.md](02-log-source-and-agent-integration.md) | Per-host agent rollout and the network troubleshooting required to get there |

---

*Part of the [MedNet Enterprise Lab](../README.md), an Enterprise Healthcare IT Infrastructure & Security Operations home lab.*
