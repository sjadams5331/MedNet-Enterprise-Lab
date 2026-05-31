# 01 — AD Integration

## Overview

This document covers how `mednet-fs01` was joined to the `mednet.lab` Active Directory domain and configured to authenticate users and resolve groups against AD. As a domain member, the file server uses the same Kerberos infrastructure as the domain-joined Windows machines — staff access shares with their existing AD identities, and no separate local accounts are maintained on the server.

This AD integration is the foundation the rest of the module is built on: the hospital share structure ([02-share-structure.md](02-share-structure.md)) and the AD group-based permissions ([03-permissions-and-acls.md](03-permissions-and-acls.md)) both depend on the server being able to resolve AD users and their group memberships. The server participates as a standard domain member, **not** a domain controller.

---

## Authentication Architecture

The file server uses two complementary AD protocols, each handling a distinct job:

| Protocol | Role |
|---|---|
| **Kerberos** | Interactive authentication — users access shares using AD tickets; on domain-joined clients no password is re-entered |
| **LDAP** | Directory lookups — resolving AD user and group information so share permissions can be enforced |

This is the same Kerberos infrastructure the domain-joined Windows endpoints use. Authentication happens on the file server (which is domain-joined), so a client does not itself need to be domain-joined to be authenticated against AD — it only changes whether the experience is single-sign-on or credential-prompted.

---

## Prerequisites

The following were confirmed before attempting the domain join:

- Domain controller `dc01.mednet.lab` running and reachable at `192.168.56.10`
- Ports `389` (LDAP) and `445` (SMB) open on the DC
- The file server's DNS resolver pointing at the DC
- System time synchronized (Kerberos requires clocks within 5 minutes)
- AD security groups already created in `OU=Security Groups,DC=mednet,DC=lab` (see [MedNet-ActiveDirectory/01-domain-design.md](../../01-MedNet-ActiveDirectory/docs/01-domain-design.md))

---

## DNS Configuration

The file server's resolver was pointed at the domain controller so that AD service records (`_ldap`, `_kerberos`) resolve and the domain can be discovered:

```
# /etc/resolv.conf
domain mednet.lab
search mednet.lab
nameserver 192.168.56.10
```

---

## Package Installation

The packages required to join and authenticate against AD were installed:

```bash
apt install realmd sssd sssd-tools adcli krb5-user packagekit samba-common-bin -y
```

| Package | Purpose |
|---|---|
| `realmd` | Domain discovery and join orchestration |
| `sssd` / `sssd-tools` | System Security Services Daemon and utilities — AD authentication |
| `adcli` | Active Directory command-line interface |
| `krb5-user` | Kerberos client utilities (`kinit`, `klist`) |
| `packagekit` | Dependency required by `realmd` |
| `samba-common-bin` | Samba tools including `net ads` |

During installation the Kerberos configuration prompt was answered as follows:

| Prompt | Value |
|---|---|
| Default Kerberos realm | `MEDNET.LAB` |
| Kerberos server | `192.168.56.10` |
| Administrative server | `192.168.56.10` |

> **Note:** The Kerberos realm is entered in uppercase (`MEDNET.LAB`). This is a Kerberos convention, not a typo — the realm name is case-sensitive and AD presents it uppercase.

![Kerberos realm configuration dialog](../screenshots/01-ad-integration.md_01.png)

---

## smb.conf — Domain Member Configuration

Before joining, the `[global]` section of `/etc/samba/smb.conf` was configured to operate as an AD domain member. The directives below are the ones that govern AD integration and identity mapping; the SMB transport-hardening directives (signing, minimum protocol, NTLM) are documented in [04-security-hardening.md](04-security-hardening.md).

```ini
[global]
   workgroup = MEDNET
   realm = MEDNET.LAB
   server string = MedNet File Server
   security = ADS
   kerberos method = secrets and keytab
   log file = /var/log/samba/log.%m
   max log size = 50

   idmap config * : backend = tdb
   idmap config * : range = 10000-99999
   idmap config MEDNET : backend = ad
   idmap config MEDNET : range = 100000-999999
```

> **Note — `security = ADS`:** This tells Samba to operate as a domain member that authenticates against Active Directory, rather than maintaining its own local user database. This single setting was also the fix for an early join failure (see below).

> **Note — `idmap config`:** The `*` (default) domain uses the `tdb` backend for any non-domain SIDs, while the `MEDNET` domain uses the `ad` backend so that UID/GID values are derived consistently from AD itself. Without a correct `idmap config` block, `testparm` reports errors and group resolution does not work. A single-character typo here (`MEDENT` instead of `MEDNET`) produced a persistent, misleading error that took careful line-by-line review to spot — the domain name in these lines must match the `workgroup` exactly.

---

## Domain Join

The join was not straightforward, and the path to success is worth preserving because the failures are common and the error messages are misleading.

**Attempt 1 — `realm join`** failed with a Kerberos encryption negotiation error (`Couldn't set password for computer account: Message stream modified`). This initially looked like a time-sync problem but was not — it was the encryption types being negotiated for the computer account password.

**Attempt 2 — `net ads join`** failed with `This operation is only allowed for the PDC`. This was resolved by correcting `smb.conf` to use `security = ADS` (above).

**Attempt 3 — `net ads join`** then failed with `No logon servers available`, despite ports `135`, `389`, and `445` all being open and DC services confirmed running.

**Working solution** — pre-authenticate as the domain administrator to obtain a Kerberos ticket, then join using that ticket:

```bash
kinit Administrator@MEDNET.LAB
net ads join -k
```

> **Why this worked:** `kinit` obtains a Kerberos TGT for the administrator account up front, and `net ads join -k` uses that existing ticket rather than negotiating credentials during the join. This sidesteps the encryption-negotiation failure that broke the other two methods. This same `kinit` + `net ads join -k` approach is the reliable fallback used elsewhere in the lab for joining Linux hosts to the Server 2025 domain.

![Successful domain join output](../screenshots/01-ad-integration.md_02.png)

---

## Validation

After joining, the configuration and AD integration were verified.

**Configuration check** — `testparm` confirmed a valid config and domain-member role:

```bash
testparm
```

```
Loaded services file OK.
Server role: ROLE_DOMAIN_MEMBER
```

**Service restart:**

```bash
systemctl restart smbd nmbd
systemctl status smbd
```

**Join and identity resolution** — these confirm the server can not only see the domain but resolve AD users *and their group memberships*, which is what share permissions depend on:

```bash
net ads testjoin                  # "Join is OK"
wbinfo -u | head                  # AD users visible to winbind
wbinfo -g | grep -i clinical      # the clinical security groups
getent passwd l.nguyen            # NSS resolves the AD user
getent group Clinical-Nursing     # NSS resolves the AD group
id l.nguyen                       # user's group memberships, incl. Clinical-Nursing
```

> **The critical check is `id l.nguyen`.** Its output must list the user's AD security group (e.g. `Clinical-Nursing`). If the group does not appear here, the corresponding share will deny that user even when the share is configured correctly — making this the first thing to check when troubleshooting access later.

| | |
|---|---|
| ![testparm domain member role](../screenshots/01-ad-integration.md_03.png) | ![id command showing AD group resolution](../screenshots/01-ad-integration.md_04.png) |

---

## Related Documents

| Document | Description |
|---|---|
| [02-share-structure.md](02-share-structure.md) | Hospital share layout, on-disk structure, and `smb.conf` share definitions |
| [03-permissions-and-acls.md](03-permissions-and-acls.md) | AD group-to-share mapping, POSIX ACLs, and access control |
| [04-security-hardening.md](04-security-hardening.md) | SMB signing, protocol hardening, firewall, and SSH hardening |
| [MedNet-ActiveDirectory/01-domain-design.md](../../01-MedNet-ActiveDirectory/docs/01-domain-design.md) | AD OU structure, security groups, and user accounts |
