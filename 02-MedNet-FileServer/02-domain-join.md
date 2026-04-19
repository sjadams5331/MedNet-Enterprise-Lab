# 02 — Domain Join
 
## Overview
 
This document covers joining `mednet-fs01` to the `mednet.lab` Active Directory domain. Domain membership allows Samba shares to authenticate users via Kerberos and enforce AD group-based access control without managing separate local credentials.
 
---
 
## Authentication Architecture
 
The file server uses two complementary protocols for AD integration:
 
| Protocol | Role |
|---|---|
| **Kerberos** | Interactive authentication — users access shares using AD tickets, no password re-entry required |
| **LDAP** | Directory lookups — resolving AD user and group information for share permissions |
 
This is the same Kerberos infrastructure used by domain-joined Windows machines. The file server participates as a standard domain member, not a domain controller.
 
---
 
## Prerequisites
 
Before joining the domain the following must be confirmed:
 
- AD server is running and reachable at `192.168.56.10`
- Port `389` (LDAP) and port `445` (SMB) are open on the AD server
- DNS on the file server points to the AD server
- System time is synchronized (Kerberos requires clocks within 5 minutes)
---
 
## DNS Configuration
 
The file server's DNS resolver was pointed at the AD server to enable domain discovery:
 
```
/etc/resolv.conf
```
 
```
domain mednet.lab
search mednet.lab
nameserver 192.168.56.10
```
 
---
 
## Required Package Installation
 
The following packages were installed to support domain join:
 
```bash
apt install realmd sssd sssd-tools adcli krb5-user packagekit samba-common-bin -y
```
 
| Package | Purpose |
|---|---|
| `realmd` | Domain discovery and join orchestration |
| `sssd` | System Security Services Daemon — AD authentication |
| `sssd-tools` | SSSD management utilities |
| `adcli` | Active Directory command-line interface |
| `krb5-user` | Kerberos client utilities |
| `samba-common-bin` | Samba tools including `net ads` |
 
During installation, the Kerberos configuration prompt was answered as follows:
 
| Prompt | Value |
|---|---|
| Default Kerberos realm | `MEDNET.LAB` |
| Kerberos server | `192.168.56.10` |
| Administrative server | `192.168.56.10` |
 
---
 
## Screenshot — Kerberos Configuration Prompt
 
<!-- Add screenshot: Kerberos realm configuration dialog during package install -->
 
---
 
## smb.conf Global Configuration
 
Before joining, `/etc/samba/smb.conf` was configured with the correct domain settings:
 
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
   server signing = mandatory
   client signing = mandatory
   server min protocol = SMB2
   ntlm auth = no
```
 
> **Note:** `security = ADS` tells Samba to operate as a domain member using Active Directory for authentication rather than managing its own user database.
 
---
 
## Domain Join
 
A Kerberos ticket was obtained first, then used to perform the domain join:
 
```bash
kinit Administrator@MEDNET.LAB
klist
net ads join -k
```
 
The `kinit` command authenticates to the KDC and stores a Kerberos ticket. The `-k` flag on `net ads join` uses that existing ticket rather than prompting for a password again.
 
---
 
## Screenshot — Kerberos Ticket (klist output)
 
<!-- Add screenshot: klist showing valid ticket for Administrator@MEDNET.LAB -->
 
---
 
## Screenshot — Successful Domain Join
 
<!-- Add screenshot: net ads join output showing "Joined 'MEDNET-FS01' to dns domain 'mednet.lab'" -->
 
---
 
## Join Verification
 
Domain membership was verified with:
 
```bash
net ads info
```
 
**Expected output confirms:**
 
| Field | Value |
|---|---|
| LDAP server | `192.168.56.10` |
| LDAP server name | `dc01.mednet.lab` |
| Realm | `MEDNET.LAB` |
| Bind Path | `dc=MEDNET,dc=LAB` |
| KDC server | `192.168.56.10` |
 
---
 
## Screenshot — net ads info Output
 
<!-- Add screenshot: net ads info showing full domain membership details -->
 
---
 
## AD Verification
 
The computer account `MEDNET-FS01` was confirmed in Active Directory Users and Computers under the `Computers` container.
 
---
 
## Screenshot — MEDNET-FS01 in Active Directory
 
<!-- Add screenshot: ADUC showing MEDNET-FS01 in the Computers container -->
 
---
 
## Troubleshooting Notes
 
During the domain join process several issues were encountered and resolved:
 
| Issue | Cause | Resolution |
|---|---|---|
| `No such realm found` | DNS not pointing to AD server | Updated `/etc/resolv.conf` to use `192.168.56.10` as nameserver |
| `This operation is only allowed for the PDC` | `smb.conf` missing `security = ADS` | Replaced default config with correct ADS member configuration |
| `Couldn't set password for computer account: Message stream modified` | Kerberos encryption negotiation failure | Used `kinit` to pre-authenticate and `net ads join -k` instead of `realm join` |
| `DNS update failed: NT_STATUS_UNSUCCESSFUL` | File server couldn't auto-register DNS record | Non-blocking — DNS record can be added manually on AD server if needed |
 
> These troubleshooting steps are documented here as they represent real-world scenarios encountered when joining Linux systems to Windows AD environments — a common task in enterprise IT.
 
---
 
## Related Documents
 
| Document | Description |
|---|---|
| [01-samba-installation.md](01-samba-installation.md) | Samba installation and initial setup |
| [03-share-configuration.md](03-share-configuration.md) | Hospital share structure and AD group permissions |
| [04-security-hardening.md](04-security-hardening.md) | SMB signing, firewall, SSH hardening |
