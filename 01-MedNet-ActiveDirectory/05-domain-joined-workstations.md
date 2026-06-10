# Domain-Joined Workstations

> Three departmental endpoints — clinical, administrative, and IT — joined to `mednet.lab` and authenticating against the domain controller, establishing the workstation fleet that every downstream module is demonstrated against.

## Overview

A directory service is only as meaningful as the endpoints that consume it. This document records the three workstations provisioned in MedNet and confirms each is a fully joined member of the `mednet.lab` domain, authenticating its assigned persona against `dc01.mednet.lab` and receiving the policies catalogued in [04-security-hardening.md](04-security-hardening.md).

The intent here is scoped deliberately. This module establishes that the fleet **exists**, is **centrally authenticated**, and is **policy-managed** — the prerequisites for everything else in MedNet. The day-to-day activity these endpoints generate (mapped share access on the file server, ticket submission in the helpdesk, monitoring and security-agent check-ins) is demonstrated in the modules that own that telemetry rather than duplicated here. See [Where These Endpoints Appear Next](#where-these-endpoints-appear-next).

The fleet is intentionally mixed — two Windows editions and one Linux desktop — mirroring how a real mid-size hospital runs a heterogeneous estate. Central authentication against a single directory, regardless of operating system, is the control that satisfies HIPAA's information-access-management and audit-controls expectations: every login on every platform resolves to a named domain identity and is recorded on the domain controller.

---

## Part 1 — Endpoint Inventory

| Hostname | Operating System | Persona | Department / OU | Host-Only IP |
|---|---|---|---|---|
| `WS-CLIN-01` | Windows 11 Enterprise | `l.nguyen` | Clinical (Nursing) | `192.168.56.101` |
| `WS-ADMIN-01` | Windows 10 Enterprise LTSC 2021 | `k.booth` | Administrative (HR) | `192.168.56.109` |
| `WS-IT-01` | Ubuntu 24.04 LTS Desktop (GNOME) | `a.turner` | IT (Staff) | `192.168.56.103` |

All three reside on the host-only network (`vboxnet0`, `192.168.56.0/24`) with a secondary NAT adapter for internet access, and are placed in dedicated departmental sub-OUs under `OU=Workstations,OU=MedNet,DC=mednet,DC=lab` so that machine-scoped policy applies by department.

### Operating System Selection Rationale

| Endpoint | Choice | Why |
|---|---|---|
| `WS-CLIN-01` | Windows 11 Enterprise | Clinical floors need a current, fully supported OS with ongoing patching and modern hardware compatibility — the realistic baseline for active patient-facing endpoints. |
| `WS-ADMIN-01` | Windows 10 LTSC 2021 | Administrative departments (billing, records, front desk) routinely run LTSC builds, where fixed-function legacy applications favor stability over feature churn. Including LTSC demonstrates an understanding of *why* real healthcare fleets are mixed rather than uniformly current. |
| `WS-IT-01` | Ubuntu 24.04 LTS Desktop | The IT engineering bench — scripting, SSH, and troubleshooting. Joining a Linux desktop to AD demonstrates the most transferable identity-management skill of the three and reflects how an IT staffer realistically works. |

### Verification

The departmental OU layout and the three pre-staged computer objects are confirmed in Active Directory Users and Computers.

| | |
|---|---|
| ![ADUC showing the Workstations OU with Clinical, Administrative, and IT sub-OUs](../screenshots/05-domain-joined-workstations.md_01.png) | ![The three computer objects WS-CLIN-01, WS-ADMIN-01, WS-IT-01 in their departmental OUs](../screenshots/05-domain-joined-workstations.md_02.png) |

---

## Part 2 — Pre-Join Preparation

All three endpoints share a common baseline before any join is attempted. In a dual-adapter VirtualBox environment this preparation is the source of nearly every avoidable join failure.

### Pre-Staging Computer Accounts

Computer objects are created in the target OU *before* the join, rather than letting the join drop them into the default `CN=Computers` container (which cannot have GPOs linked to it). Pre-staging guarantees machine-scoped policy such as `Workstation-Baseline-Policy` applies on the first boot after join, and keeps the build free of "wrong OU" cleanup.

```powershell
New-ADComputer -Name "WS-CLIN-01"  -Path "OU=Clinical,OU=Workstations,OU=MedNet,DC=mednet,DC=lab"
New-ADComputer -Name "WS-ADMIN-01" -Path "OU=Administrative,OU=Workstations,OU=MedNet,DC=mednet,DC=lab"
New-ADComputer -Name "WS-IT-01"    -Path "OU=IT,OU=Workstations,OU=MedNet,DC=mednet,DC=lab"
```

### Network and DNS Baseline

Every VM carries two adapters — a NAT adapter for internet access and a host-only adapter (`vboxnet0`) for internal lab traffic. DNS must be pinned carefully or the NAT adapter registers competing records in `mednet.lab` and causes intermittent, round-robin resolution failures. The following baseline is applied to all three endpoints:

| Setting | Configured Value | Rationale |
|---|---|---|
| Primary DNS | `192.168.56.10` (DC only, no fallback) | All domain lookups resolve against the DC. |
| NAT-adapter DNS registration | Disabled | Prevents the NAT interface from registering `10.0.2.x` records in `mednet.lab`. |
| IPv6 | Disabled on both adapters | Eliminates dual-stack resolution ambiguity in the lab. |
| Time source | Domain controller | Kerberos rejects clock skew greater than five minutes. |

### Hostname

Each machine is renamed to exactly match its pre-staged AD object before joining. On Windows this happens after OOBE; on Ubuntu it is set during installation. Joining with a mismatched name creates a second, orphaned object in the default container while the pre-staged object sits unused.

```powershell
# Windows — elevated prompt, after OOBE and before domain join
Rename-Computer -NewName "WS-CLIN-01" -Restart
```

### Verification

Pre-flight resolution and adapter state are confirmed before joining: a clean `ipconfig /all` on Windows (correct IP marked *Preferred*, no Duplicate flag, DNS pinned to the DC) and a clean resolver state on Ubuntu.

| | |
|---|---|
| ![Windows ipconfig /all showing pinned DNS and no duplicate address](../screenshots/05-domain-joined-workstations.md_03.png) | ![Ubuntu resolvectl status showing the DC as the resolver for mednet.lab](../screenshots/05-domain-joined-workstations.md_04.png) |

---

## Part 3 — Windows Workstation Join

`WS-CLIN-01` and `WS-ADMIN-01` follow an identical sequence; only the hostname, IP, and persona user differ.

### Join Sequence

1. Complete OOBE, install Guest Additions, reboot.
2. `Rename-Computer` to match the pre-staged object, reboot.
3. Pin DNS to `192.168.56.10`; disable IPv6 and NAT-adapter DNS registration.
4. Confirm the domain locator and time sync before joining:

```cmd
nltest /dsgetdc:mednet.lab
ping dc01.mednet.lab
w32tm /query /status
```

5. Join via **System Properties → Computer Name → Member of Domain → `mednet.lab`**, authenticating as `MNHS.LocalAdmin`. Reboot.
6. Log in as the persona's domain user and confirm policy application.

### Verification

Domain membership is confirmed in System Properties, and per-user policy is verified with `gpresult /r`:

```cmd
gpresult /r
```

For `WS-CLIN-01` (`l.nguyen`), the applied set includes `Default Domain Policy`, `Workstation-Baseline-Policy`, and `Clinical-Workstation-Policy`. For `WS-ADMIN-01` (`k.booth`), `Administrative-User-Policy` applies in place of the clinical policy. This confirms endpoint-side enforcement of the controls defined in [04-security-hardening.md](04-security-hardening.md).

| | |
|---|---|
| ![System Properties showing membership in mednet.lab](../screenshots/05-domain-joined-workstations.md_05.png) | ![gpresult /r on WS-CLIN-01 as l.nguyen showing the Clinical policy applied](../screenshots/05-domain-joined-workstations.md_06.png) |

![gpresult /r on WS-ADMIN-01 as k.booth showing the Administrative policy applied](../screenshots/05-domain-joined-workstations.md_07.png)

---

## Part 4 — Linux Workstation Join (WS-IT-01)

Joining an Ubuntu desktop to AD is the most involved of the three and demonstrates the most transferable Linux identity skill. The endpoint authenticates the same domain users against the same DC, with login presented at the GNOME (GDM) screen.

### Why `net ads join` instead of `realm join`

The standard `realm join` path failed against Windows Server 2025. `adcli` negotiated a DES encryption type that the modern DC rejected, surfacing as a `Message stream modified` Kerberos error; a follow-up `realm join --membership-software=samba` then failed on Kerberos FAST armoring. Documenting the failure path matters here — it reflects a real interoperability quirk between current Ubuntu tooling and Server 2025, and the working solution is a deliberate Samba-based join rather than the textbook one-liner.

### Working Join Path

The reliable sequence configures Samba explicitly, joins with `net ads`, then wires up SSSD by hand. The critical line is `client use kerberos = desired` (not `required`), which lets the client negotiate a modern enctype instead of forcing the rejected default.

```ini
# /etc/samba/smb.conf
[global]
   workgroup = MEDNET
   realm = MEDNET.LAB
   security = ADS
   client use kerberos = desired
```

```bash
sudo net ads join -U Administrator
```

SSSD is then configured for ID mapping and offline login:

```ini
# /etc/sssd/sssd.conf
[domain/mednet.lab]
   use_fully_qualified_names = False
   fallback_homedir = /home/%u
   ldap_id_mapping = True
   cache_credentials = True
```

```bash
sudo pam-auth-update --enable mkhomedir
```

### Verification

The join is confirmed by resolving a domain user through NSS and checking group membership; the GDM screen then accepts the domain login directly.

| | |
|---|---|
| ![net ads join completing successfully against mednet.lab](../screenshots/05-domain-joined-workstations.md_08.png) | ![getent passwd a.turner and id a.turner showing UID, GID, and IT-Staff membership](../screenshots/05-domain-joined-workstations.md_09.png) |

![GNOME (GDM) login as the domain user a.turner on WS-IT-01](../screenshots/05-domain-joined-workstations.md_10.png)

---

## Part 5 — Centralized Authentication

The point of the fleet is a single source of truth for identity. With all three endpoints joined, every interactive login — Windows or Linux — produces a `Logon` event (Event ID `4624`) in the domain controller's Security log, attributable to a named domain account. This is the audit-trail foundation that the SIEM module later builds detection content on top of.

| | |
|---|---|
| ![DC Security log showing Event 4624 logons from WS-CLIN-01, WS-ADMIN-01, and WS-IT-01](../screenshots/05-domain-joined-workstations.md_11.png) | |

---

## Where These Endpoints Appear Next

These workstations are the actors for the rest of MedNet. Their provisioning is recorded here once; the activity they generate is demonstrated in the module that owns it:

| Activity | Demonstrated In |
|---|---|
| Mapped departmental share access (Clinical / Administrative / IT) and minimum-necessary enforcement | [02-MedNet-FileServer](../../02-MedNet-FileServer) |
| End-user ticket submission and helpdesk workflow | [03-MedNet-TicketingSystem](../../03-MedNet-TicketingSystem) |
| Performance and availability agent check-ins | [04-MedNet-NetworkMonitoring](../../04-MedNet-NetworkMonitoring) |
| Security-event collection and endpoint detections | [06-MedNet-SIEM](../../06-MedNet-SIEM) |

---

## Related Documents

| Document | Description |
|---|---|
| [01-domain-design.md](01-domain-design.md) | Domain, OU, and naming-convention design that these endpoints conform to. |
| [04-security-hardening.md](04-security-hardening.md) | The GPO and account-hardening controls enforced on the joined fleet. |
| [README.md](../README.md) | Active Directory module overview and documentation index. |
