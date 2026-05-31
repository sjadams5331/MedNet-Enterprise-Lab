# MedNet File Server

> **AD-Integrated Samba File Server for mednet.lab**
> A Debian-based Samba file server joined to the `mednet.lab` Active Directory domain, providing department-scoped SMB shares whose access is governed entirely by AD security groups. Models the file services tier of a hospital IT environment, with access compartmentalized along clinical, administrative, and IT lines in line with HIPAA minimum-necessary principles.

---

## Overview

`mednet-fs01` is a headless Debian 12 server running Samba as a domain member of `mednet.lab`. Rather than maintaining its own user database, it authenticates staff against Active Directory over Kerberos and enforces access through AD security groups — the same identities used everywhere else in the lab. Department shares are presented as four category shares (Clinical, Administrative, IT, Shared), each admitting only the AD groups appropriate to it.

This VM is part of the broader **MedNet Enterprise Lab** — a simulated hospital IT infrastructure built for portfolio and skills-development purposes.

---

## Environment Details

| Setting | Value |
|---|---|
| Hostname | `mednet-fs01` |
| FQDN | `mednet-fs01.mednet.lab` |
| IP Address | `192.168.56.20` |
| Operating System | Debian 12 (Bookworm) — Headless Server |
| Domain | `mednet.lab` |
| Role | Samba File Server — Domain Member |
| Samba Version | 4.17.12-Debian |

---

## Prerequisites

The following lab components must be running and reachable before this server is built or used:

| Dependency | Details |
|---|---|
| AD Domain Controller | `dc01.mednet.lab` at `192.168.56.10` |
| Domain | `mednet.lab` with functional Kerberos |
| AD Security Groups | Department groups in `OU=Security Groups,DC=mednet,DC=lab` |
| Internal CA | `MedNet-RootCA` available on the DC |
| VirtualBox Host-Only Network | Shared with all lab VMs on `192.168.56.0/24` |

---

## Architecture

The file server participates as a standard domain member — not a domain controller. Authentication happens on the server itself via Kerberos, and AD users and groups are resolved through winbind. Access to each share is determined by AD group membership, so permissions are managed centrally in Active Directory rather than locally on the server.

```
   AD Domain Controller                 File Server (domain member)
   dc01.mednet.lab                       mednet-fs01.mednet.lab
   192.168.56.10                         192.168.56.20
        │                                       │
        │  Kerberos (auth) + LDAP (lookups)     │
        └───────────────────────────────────────┘
                          │
        AD security groups govern share access:
        Clinical-*  →  [Clinical]    Admin-*  →  [Administrative]
        IT-Staff    →  [IT]          Domain Users  →  [Shared]
```

---

## Hospital Share Structure

Four department-category shares are served, each gated by the AD security groups belonging to that category. Individual departments exist as subdirectories within each share.

| Share | Scope | AD Groups Permitted | On-disk Path |
|---|---|---|---|
| `Clinical` | Clinical — PHI (Restricted) | `Clinical-Physicians`, `Clinical-Nursing`, `Clinical-Pharmacy` | `/srv/samba/clinical` |
| `Administrative` | Administrative (Restricted) | `Admin-HR`, `Admin-Finance`, `Admin-Reception` | `/srv/samba/administrative` |
| `IT` | IT Department (Restricted) | `IT-Staff` | `/srv/samba/it` |
| `Shared` | Hospital-wide Shared Resources | `Domain Users` | `/srv/samba/shared` |

Access is enforced in two layers: the share-level `valid users` gate (who may connect) and filesystem POSIX ACLs (what a connected user may read or write). The category boundary is the primary HIPAA minimum-necessary control — staff in one category cannot reach another category's share.

---

## Repository Structure

```
02-MedNet-FileServer/
├── README.md                       ← You are here
├── docs/
│   ├── 01-ad-integration.md        ← Domain join, Kerberos, AD identity resolution
│   ├── 02-share-structure.md       ← Share layout, smb.conf definitions, AD-group gating
│   ├── 03-permissions-and-acls.md  ← POSIX ACLs, read/write control, allow/deny demo
│   ├── 04-security-hardening.md    ← SMB signing, protocol hardening, firewall, SSH
│   ├── 05-backup-and-recovery.md   ← Backup method, retention, tested restore
│   └── 06-storage-and-quotas.md    ← Storage layout and per-group disk quotas
└── screenshots/
```

---

## Skills Demonstrated

- Linux server administration on headless Debian
- Joining a Linux host to an Active Directory domain via Kerberos (`net ads join`)
- AD identity resolution on Linux with winbind and `idmap` (rid backend)
- Samba domain-member configuration and SMB share design
- Group-based access control mapping AD security groups to file resources
- POSIX ACLs as an NTFS-equivalent permission layer
- SMB protocol and host hardening, and host-based firewalling
- Backup, recovery, and storage/quota management

---

## Documentation

| Document | Description |
|---|---|
| [01-ad-integration.md](docs/01-ad-integration.md) | Domain join, Kerberos authentication, and AD identity resolution |
| [02-share-structure.md](docs/02-share-structure.md) | Share layout, on-disk structure, and `smb.conf` share definitions |

*Permissions & ACLs, security hardening, backup & recovery, and storage & quotas are added as each phase is completed.*

---

*Part of the [MedNet Enterprise Lab](../README.md) — Enterprise Healthcare IT Infrastructure & Security Operations Home Lab*
