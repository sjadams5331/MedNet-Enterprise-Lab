# 05 — Domain-Joined Workstations

## Overview

This document covers the deployment and Active Directory domain integration of the MedNet Enterprise Lab endpoint fleet — three workstations representing the clinical, administrative, and IT engineering personas of a hospital environment. Each workstation runs a deliberately different operating system to demonstrate cross-platform endpoint management, and each is joined to the `mednet.lab` domain so that authentication, authorization, group policy, and audit are all enforced centrally from the domain controller rather than configured by hand on each machine.

Bringing endpoints under domain control is the practical foundation of the HIPAA Security Rule's technical safeguards. Centralized identity is what allows access control (§164.312(a)(1)), audit controls (§164.312(b)), and person-or-entity authentication (§164.312(d)) to be applied consistently to every device that could touch electronic protected health information (ePHI).

This document is scoped to workstation provisioning and domain membership. The group policies these endpoints receive are detailed in [02-gpo-configuration.md](02-gpo-configuration.md) and [04-security-hardening.md](04-security-hardening.md); endpoint monitoring and security agents are documented in the monitoring and SIEM modules, where that telemetry converges.

---

## Part 1 — Endpoint Inventory

The fleet is intentionally heterogeneous. A production hospital runs a mix of operating systems — locked-down clinical devices, standard administrative desktops, and engineering workstations — and demonstrating a single, consistent identity and access model across all of them is the core skill this section is built to show.

| Hostname | Operating System | Persona | Primary User | IP Address | Computer Object OU |
|---|---|---|---|---|---|
| WS-CLIN-01 | Windows 11 Enterprise | Clinical (nurse station) | l.nguyen (Clinical-Nursing) | 192.168.56.101 | Workstations > Computers |
| WS-ADMIN-01 | Windows 10 Enterprise LTSC 2021 | Administrative (front desk) | k.booth (Admin-HR) | 192.168.56.109 | Workstations > Computers |
| WS-IT-01 | Ubuntu 24.04 LTS Desktop | IT engineering bench | a.turner (IT-Staff) | 192.168.56.103 | Workstations > Computers |

OS selection rationale:

- **Windows 10 LTSC** was chosen over standard Windows 10 because LTSC is the edition hospitals realistically deploy on fixed-function clinical and front-desk endpoints, making it a more authentic portfolio narrative.
- **Ubuntu 24.04 LTS** (rather than the newest release) reflects healthcare's standard practice of never deploying day-zero LTS releases, and it has mature documentation for SSSD/realmd AD integration.
- **Ubuntu Desktop with GNOME** was selected so the domain login is a graphical GDM session — the way a helpdesk technician would actually use the machine — which also produces a clean cross-platform login screenshot.

| | |
|---|---|
| ![Three workstation VMs running in VirtualBox](../screenshots/05-domain-joined-workstations.md_01.png) | ![ADUC showing the three computer objects in Workstations > Computers](../screenshots/05-domain-joined-workstations.md_02.png) |

---

## Part 2 — Pre-Join Preparation

All three endpoints share a common preparation baseline before any domain join is attempted. Skipping these steps is the source of nearly every join failure in a dual-adapter VirtualBox environment.

### Pre-Staging Computer Accounts

Computer objects are created in the target OU *before* the join rather than letting the join drop them into the default `CN=Computers` container. Pre-staging guarantees machine-scoped GPOs (such as `Workstation-Baseline-Policy`) apply on the first boot after join, and keeps the documentation free of "wrong OU" cleanup steps.

```powershell
New-ADComputer -Name "WS-CLIN-01"  -Path "OU=Computers,OU=Workstations,OU=MedNet,DC=mednet,DC=lab"
New-ADComputer -Name "WS-ADMIN-01" -Path "OU=Computers,OU=Workstations,OU=MedNet,DC=mednet,DC=lab"
New-ADComputer -Name "WS-IT-01"    -Path "OU=Computers,OU=Workstations,OU=MedNet,DC=mednet,DC=lab"
```

### Network and DNS Baseline

Every VM has two adapters — a NAT adapter for internet access and a host-only adapter (`vboxnet0`) for internal lab traffic. DNS must be pinned carefully or the NAT adapter will register competing records in `mednet.lab` and cause round-robin resolution failures. The following baseline is applied to all three endpoints:

| Setting | Configured Value | Rationale |
|---|---|---|
| Primary DNS | 192.168.56.10 (DC only, no fallback) | All domain lookups resolve against the DC |
| NAT adapter DNS registration | Disabled | Prevents the NAT interface from registering 10.0.2.x records in `mednet.lab` |
| IPv6 | Disabled on both adapters | Eliminates dual-stack resolution ambiguity in the lab |
| Time source | Domain controller | Kerberos rejects clock skew greater than 5 minutes |

### Hostname

Each machine is named to exactly match its pre-staged AD object before the join. On Windows this happens after OOBE and before joining; on Ubuntu it is set during installation. Joining with a mismatched name creates a second, orphaned computer object in the default container while the pre-staged object goes unused.

```powershell
# Windows — run from an elevated prompt after OOBE, before domain join
Rename-Computer -NewName "WS-CLIN-01" -Restart
```

### Verification

Pre-flight resolution and adapter state are confirmed before joining: a clean `ipconfig /all` on Windows (correct IP as *Preferred*, no Duplicate flag, DNS pinned to the DC) and a clean resolver state on Ubuntu.

| | |
|---|---|
| ![Windows ipconfig /all showing pinned DNS and no duplicate IP](../screenshots/05-domain-joined-workstations.md_03.png) | ![Ubuntu netplan config and resolvectl status showing DC as resolver](../screenshots/05-domain-joined-workstations.md_04.png) |

---

## Part 3 — Windows Workstation Join

`WS-CLIN-01` and `WS-ADMIN-01` follow an identical sequence; only the hostname, IP, and persona user differ.

### Join Sequence

1. Complete OOBE, install Guest Additions, reboot.
2. `Rename-Computer` to match the pre-staged object, reboot.
3. Pin DNS to `192.168.56.10`, disable IPv6 and NAT-adapter DNS registration.
4. Verify domain locator and time sync before joining:

```cmd
nltest /dsgetdc:mednet.lab
ping dc01.mednet.lab
w32tm /query /status
```

5. Join the domain via System Properties → Computer Name → Member of Domain → `mednet.lab`, authenticating with `MNHS.LocalAdmin`. Reboot.
6. Log in as the persona's domain user and confirm policy application.

### Verification

Domain membership is confirmed in System Properties, and per-user group policy application is verified with `gpresult /r`:

```cmd
gpresult /r
```

For `WS-CLIN-01` (`l.nguyen`), the applied set includes `Default Domain Policy`, `Workstation-Baseline-Policy`, and `Clinical-Workstation-Policy` (5-minute screen lock, Control Panel hidden, removable storage blocked). For `WS-ADMIN-01` (`k.booth`), `Administrative-User-Policy` applies (10-minute lock, CMD/PowerShell restricted). This confirms the endpoint-side enforcement of the controls catalogued in [04-security-hardening.md](04-security-hardening.md).

| | |
|---|---|
| ![System Properties showing membership in mednet.lab](../screenshots/05-domain-joined-workstations.md_05.png) | ![gpresult /r on WS-CLIN-01 as l.nguyen showing Clinical GPO applied](../screenshots/05-domain-joined-workstations.md_06.png) |

![gpresult /r on WS-ADMIN-01 as k.booth showing Administrative GPO applied](../screenshots/05-domain-joined-workstations.md_07.png)

---

## Part 4 — Linux Workstation Join (WS-IT-01)

Joining an Ubuntu desktop to AD is the most involved of the three and demonstrates the most transferable Linux identity-management skill. The endpoint authenticates domain users against the same DC and applies the same account standards, with login presented at the GNOME (GDM) screen.

### Network Configuration (netplan)

The two adapters present as `enp0s3` (NAT / Adapter 1) and `enp0s8` (host-only / Adapter 2). A common pitfall is swapping these in the YAML; the host-only interface carries the static lab address and the DC resolver.

```yaml
network:
  version: 2
  ethernets:
    enp0s3:                 # NAT — internet only, no DNS registration
      dhcp4: true
      dhcp4-overrides:
        use-dns: false
    enp0s8:                 # Host-only — lab traffic + AD DNS
      addresses: [192.168.56.103/24]
      nameservers:
        addresses: [192.168.56.10]
        search: [mednet.lab]
```

If a stale NetworkManager profile overrides netplan, or `systemd-resolved` fails to propagate the DNS server to the host-only link (`Current Scopes: none` on the link), pin the resolver directly:

```bash
sudo resolvectl dns enp0s8 192.168.56.10
sudo resolvectl domain enp0s8 '~mednet.lab'
```

### Kerberos Encryption Types

Windows Server 2025 rejects legacy DES encryption types during the computer-password exchange. If the client offers weak enctypes, the join fails with a `Message stream modified` error. The client is configured to negotiate AES only in `/etc/krb5.conf`:

```ini
[libdefaults]
    default_realm = MEDNET.LAB
    default_tkt_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96
    default_tgs_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96
    dns_lookup_realm = false
    dns_lookup_kdc = true
```

### Join

```bash
sudo apt install -y realmd sssd sssd-tools adcli samba-common-bin krb5-user packagekit
sudo realm discover mednet.lab
sudo realm join --verbose \
  --computer-ou="OU=Computers,OU=Workstations,OU=MedNet,DC=mednet,DC=lab" \
  mednet.lab -U MNHS.LocalAdmin
```

SSSD is then configured to create home directories on first login and to scope interactive access to the appropriate domain group:

```bash
sudo pam-auth-update --enable mkhomedir
sudo realm permit -g IT-Staff@mednet.lab
```

### Verification

```bash
realm list
id a.turner@mednet.lab
```

`realm list` reports `mednet.lab` as configured with the `sssd` client, and `id` returns `a.turner`'s domain group memberships including `IT-Staff`. The GDM login screen accepts `a.turner@mednet.lab` and lands in a GNOME session — the cross-platform login that visually anchors this section.

| | |
|---|---|
| ![realm list and id a.turner output confirming domain membership](../screenshots/05-domain-joined-workstations.md_08.png) | ![GNOME GDM login screen authenticating a.turner@mednet.lab](../screenshots/05-domain-joined-workstations.md_09.png) |

---

## Part 5 — Centralized Authentication Verification

The payoff of joining all three endpoints to one domain is that every login — regardless of operating system — is authenticated by the same domain controller and written to the same audit trail. On the DC, filtering the Security log for Event ID 4624 (successful logon) shows interactive logons originating from all three workstations against their respective domain users.

| Workstation | Operating System | Domain User | DC Audit Events |
|---|---|---|---|
| WS-CLIN-01 | Windows 11 Enterprise | l.nguyen | 4624 (logon), 4768 (Kerberos AS-REQ) |
| WS-ADMIN-01 | Windows 10 LTSC 2021 | k.booth | 4624, 4768 |
| WS-IT-01 | Ubuntu 24.04 Desktop | a.turner | 4624, 4768 |

Three operating systems, one identity and audit model. This consolidated authentication trail is the endpoint-side complement to the audit policy defined in [04-security-hardening.md](04-security-hardening.md), and these events are forwarded to the Wazuh SIEM for correlation and retention.

![DC Security log filtered to Event ID 4624 showing logons from all three workstations](../screenshots/05-domain-joined-workstations.md_10.png)

---

## Future Enhancements

The following endpoint measures are planned for later phases of the lab:

- **Endpoint monitoring and security agents** — Zabbix and Wazuh agents deployed to all three workstations. These are documented in the monitoring and SIEM modules, where performance and security telemetry converge, rather than duplicated here.
- **Full-disk encryption** — BitLocker on the Windows endpoints (with recovery key escrow to AD) and LUKS on Ubuntu, to satisfy encryption-at-rest expectations for devices that may cache ePHI.
- **Department-scoped GPO targeting** — finer-grained policy via Clinical / Administrative / IT sub-OU links for endpoint-specific configuration beyond the current baseline.

---

## Related Documents

| Document | Description |
|---|---|
| [01-domain-design.md](01-domain-design.md) | OU structure, naming conventions, hospital org model |
| [02-gpo-configuration.md](02-gpo-configuration.md) | GPO design and the policies these endpoints receive |
| [04-security-hardening.md](04-security-hardening.md) | Domain audit policy and endpoint hardening controls |
| [MedNet-FileServer](../02-MedNet-FileServer/README.md) | AD group-based share access exercised from these workstations |
| [MedNet-SIEM](../06-MedNet-SIEM/README.md) | Forwarding and correlation of endpoint authentication events |
