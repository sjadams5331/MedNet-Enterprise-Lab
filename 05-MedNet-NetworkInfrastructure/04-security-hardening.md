# Security Hardening

## Overview

This document covers the hardening decisions applied on top of the base firewall and VLAN configuration described in `02-firewall-and-routing-configuration.md` and `03-vlan-and-inter-vlan-routing.md`. In keeping with this repository's documentation philosophy, items here are split plainly into what is implemented and verified today versus what is deliberately deferred, along with the reasoning behind each decision.

---

## Implemented

### Lateral Movement Blocking (Clinical &harr; Admin)

Clinical (VLAN 20) and Admin (VLAN 30) cannot reach each other. Both interfaces carry an explicit Block rule against the `RFC1918_Private` alias, placed after the specific allow rules for the domain controller, file server, and ticketing portal, and before a final internet-bound pass. See `02-firewall-and-routing-configuration.md` for the full rule breakdown.

This is treated as a hardening decision rather than a default: keeping clinical systems isolated from administrative and business systems is one of the more meaningful segmentation boundaries a healthcare-modeled network can demonstrate, and it is enforced as a visible, intentional rule rather than left as an accidental side effect of what other rules happen to be absent.

### DMZ Outbound Restriction

The DMZ (VLAN 50, osTicket) is permitted exactly one outbound path: to the domain controller, TCP port 636 (LDAPS), for staff authentication. No default internet pass exists on this interface; pfSense's implicit deny-all handles everything else. Every other VLAN is granted inbound access to the DMZ on port 443 only, so the ticketing portal is broadly reachable for ticket submission while having almost no ability to reach anywhere else if it were ever compromised.

### LAN (Servers) Outbound Narrowing

The domain controller and file server previously shared LAN's stock default-allow rule. This has been replaced with a scoped rule set:

| # | Action | Destination | Port | Purpose |
|---|---|---|---|---|
| 1 | Block | `RFC1918_Private` alias | Any | No lateral reach into other VLANs |
| 2 | Pass | Any | TCP+UDP / 53 | DNS forwarding |
| 3 | Pass | Any | TCP / 80 | Windows Update, CRL checks |
| 4 | Pass | Any | TCP / 443 | Windows Update, OCSP, general HTTPS |
| 5 | Pass | Any | UDP / 123 | NTP time sync |

Unlike the client VLANs, LAN does not need explicit allow rules for the DC or file server themselves, since most traffic to those hosts is inbound, initiated by other segments, and handled automatically by pfSense's state table. The block rule can therefore sit first in the list rather than needing specific allows placed above it.

![LAN outbound rule set showing the RFC1918 block followed by the four scoped pass rules](../screenshots/04-security-hardening_01.png)

**A pre-existing issue surfaced during testing, unrelated to this change:** the domain controller's configured DNS forwarders (`192.168.1.1`, `43.168.1.1`) were not genuinely reachable addresses, likely stray values auto-populated at some point and never corrected. External DNS resolution had almost certainly been broken since before this hardening pass began; it simply had never been tested until this rule set required verifying outbound DNS explicitly. Corrected via `Set-DnsServerForwarder` to known-reachable public resolvers. Documented here rather than in the routing doc since it was discovered as a direct result of hardening verification, not the VLAN migration work itself.

Each of the four pass rules (HTTPS, DNS via raw port and via the corrected forwarder, NTP) was independently tested and confirmed working post-change, along with a check that core AD services (DNS, Netlogon, NTDS) remained healthy.

![DNS Manager forwarders tab showing the corrected, reachable public resolvers](../screenshots/04-security-hardening_02.png)

### pfSense Administrative Access

Several settings under System &rarr; Advanced &rarr; Admin Access were reviewed and hardened:

- **Protocol:** HTTPS only. No plaintext HTTP listener; the WebGUI redirect rule remains enabled, forcing any port-80 request to HTTPS automatically. HSTS remains enabled, forcing the browser to HTTPS for all future requests to the firewall's FQDN.
- **Login autocomplete:** disabled. Firewall admin credentials are no longer eligible to be saved by the browser.
- **Login protection:** confirmed active, using pfSense's built-in defaults (threshold 30, initial blocktime 120 seconds with 1.5x escalation on repeat offenses, 30-minute detection window). This applies to both the web GUI and SSH. No addresses were added to the pass list; trusted management hosts are deliberately not exempted, since those are exactly the hosts most valuable to an attacker attempting credential-based lateral movement.
- **SSH:** enabled deliberately, in anticipation of Module 08's planned Ansible-based configuration management, rather than left on its off-by-default state indefinitely. Authentication is restricted to public-key only (SSHd Key Only set to "Public Key Only"); password-based SSH login is disabled entirely. A dedicated Ed25519 keypair was generated on WS-IT-01, the environment's designated jump host, and its public key added to the pfSense `admin` account. End-to-end key-based login was verified from WS-IT-01 before switching off password authentication, avoiding a lockout scenario (SSH is not covered by pfSense's LAN Anti-Lockout Rule, unlike the web GUI).

![pfSense Admin Access settings showing HTTPS-only, disabled autocomplete, and active login protection](../screenshots/04-security-hardening_03.png)

### CA-Signed Administrative Certificate

The pfSense GUI previously used its default self-signed certificate. A certificate signing request was generated on pfSense (private key never leaving the appliance) with Common Name `pfsense.mednet.lab`, then signed by `MedNet-RootCA` on the domain controller using the WebServer certificate template, tying this module's PKI trust back to the Active Directory Certificate Services work established in `01-MedNet-ActiveDirectory`. The signed certificate was imported and assigned as the GUI's active certificate, and a corresponding DNS A record was created, consistent with the `dc01`/`itsm01` alias pattern used elsewhere in the environment.

Browsing to `https://pfsense.mednet.lab` from a domain-joined host now resolves without a certificate warning, since those machines trust `MedNet-RootCA` automatically via Group Policy. Non-domain-joined hosts (WS-IT-01, the hypervisor host itself) continue to show an untrusted-certificate warning unless the root CA certificate is separately imported into their trust stores; this is noted as a known, low-priority gap rather than a defect, since administrative access to the GUI from those hosts is a secondary path, not the primary one.

![Browser showing a valid, CA-trusted padlock for https://pfsense.mednet.lab on a domain-joined host](../screenshots/04-security-hardening_04.png)

---

## Deferred

### OPT3 (IT/Ops) Outbound Scoping

OPT3 currently retains an unrestricted Pass/Any/Any/Any rule. This is intentional for the environment's current state: VLAN 40 is the management and monitoring tier, hosting the jump host and the future targets of Zabbix and Wazuh agent traffic across every other VLAN. Narrowing this interface to its actual required ports (Zabbix agent 10050/10051, Wazuh 1514/1515, plus DC and file-server access) is planned once agent deployment in Modules 04 and 06 exists to scope against, rather than guessing at ports that may not match the eventual real configuration.

One dependency to track when this work happens: OPT3's current broad rule also incidentally grants WS-IT-01 access to the pfSense GUI itself (`10.10.10.1:443`), a legitimate and deliberate access decision (IT/Ops staff administering firewall infrastructure). Whatever scoped rule set eventually replaces OPT3's Any/Any must explicitly retain this path, or IT/Ops will silently lose administrative access to the firewall the day that hardening work lands.

### Two-Factor Authentication for pfSense Admin Login

Native TOTP support exists in pfSense's User Manager and was evaluated but not implemented in this pass. Considered the most meaningful remaining hardening step for the admin account, since it would prevent a compromised or guessed password alone from being sufficient for access. Deferred due to time rather than any technical blocker.

### Default Admin Account

The pfSense `admin` account retains its default name. Renaming or replacing it with a differently named, admin-privileged account would remove a common assumption attackers make when targeting pfSense deployments generally. Deferred; low effort, and a reasonable candidate for the next hardening pass alongside 2FA.

### CA Trust on Non-Domain-Joined Hosts

WS-IT-01 (Ubuntu, not domain-joined) and the hypervisor host itself do not trust `MedNet-RootCA`, so the pfSense GUI's now-valid certificate still presents as untrusted from those machines. Manually importing the CA certificate into their respective trust stores would resolve this. Deferred as low priority, since these are secondary access paths to the GUI, not the primary one.

---

## Related Documents

| Document | Description |
|---|---|
| [README.md](README.md) | Network Infrastructure module overview and documentation index |
| [01-network-segmentation-design.md](01-network-segmentation-design.md) | VLAN architecture rationale and subnet design |
| [02-firewall-and-routing-configuration.md](02-firewall-and-routing-configuration.md) | Interface and firewall rule configuration |
| [03-vlan-and-inter-vlan-routing.md](03-vlan-and-inter-vlan-routing.md) | Host migration, inter-VLAN routing verification, and troubleshooting |

---

*Part of the [MedNet Enterprise Lab](../README.md), an Enterprise Healthcare IT Infrastructure & Security Operations home lab.*
