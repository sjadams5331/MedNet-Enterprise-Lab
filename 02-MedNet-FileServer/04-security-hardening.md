# 04 — Security Hardening
 
## Overview
 
This document covers the security hardening applied to `mednet-fs01` after the base Samba and domain join configuration. Hardening focuses on four areas: protocol security, authentication restrictions, host-based firewall rules, and SSH access control.
 
In a healthcare environment, file servers are subject to HIPAA Security Rule requirements for access control, audit controls, and transmission security. The measures documented here address each of these areas at the infrastructure level.
 
---
 
## SMB Protocol Hardening
 
The following settings were added to the `[global]` section of `/etc/samba/smb.conf`:
 
```ini
server signing = mandatory
client signing = mandatory
server min protocol = SMB2
ntlm auth = no
```
 
### Explanation
 
| Setting | Value | Security Benefit |
|---|---|---|
| `server signing = mandatory` | mandatory | All SMB packets must be cryptographically signed — prevents man-in-the-middle and relay attacks |
| `client signing = mandatory` | mandatory | Enforces signing on connections initiated by this server |
| `server min protocol = SMB2` | SMB2 | Disables SMB1 — eliminates EternalBlue and related vulnerabilities |
| `ntlm auth = no` | no | Disables NTLM authentication — forces Kerberos, preventing pass-the-hash attacks |
 
> **Why this matters in healthcare:** SMB relay attacks are a common lateral movement technique in ransomware campaigns. Mandatory SMB signing is a direct mitigation. SMB1 was the vector for WannaCry, which caused widespread disruption in hospital environments globally in 2017. Disabling it is a non-negotiable baseline control.
 
---
 
## Screenshot — smb.conf Global Section with Hardening Settings
 
<!-- Add screenshot: nano /etc/samba/smb.conf showing global section with signing and protocol settings -->
 
---
 
## UFW Firewall Configuration
 
`ufw` (Uncomplicated Firewall) was installed and configured to restrict inbound traffic to only required services:
 
```bash
apt install ufw -y
```
 
### Firewall Rules Applied
 
```bash
ufw allow ssh
ufw allow 445/tcp
ufw allow 139/tcp
ufw allow 137/udp
ufw allow 138/udp
ufw allow from 192.168.56.0/24
ufw enable
```
 
### Port Reference
 
| Port | Protocol | Service | Purpose |
|---|---|---|---|
| 22 | TCP | SSH | Remote administration |
| 445 | TCP | SMB | Primary Samba file sharing |
| 139 | TCP | NetBIOS Session | Legacy SMB over NetBIOS |
| 137 | UDP | NetBIOS Name | NetBIOS name resolution |
| 138 | UDP | NetBIOS Datagram | NetBIOS datagram service |
 
The `ufw allow from 192.168.56.0/24` rule permits all traffic from the host-only lab network, ensuring the AD server and future VMs can communicate with the file server without additional rules.
 
### Default Policy
 
UFW default policy is **deny incoming, allow outgoing** — any port not explicitly allowed above is blocked.
 
---
 
## Screenshot — UFW Status
 
<!-- Add screenshot: ufw status verbose showing all rules and Status: active -->
 
---
 
## SSH Hardening
 
The OpenSSH server was installed and configured to prevent direct root login:
 
```bash
apt install openssh-server -y
```
 
`/etc/ssh/sshd_config` was edited to set:
 
```
PermitRootLogin no
```
 
This forces all administrative access to go through a named user account (`sysadmin`) with `su` or `sudo` escalation, creating an audit trail for privileged actions.
 
```bash
systemctl restart sshd
```
 
---
 
## Screenshot — sshd_config PermitRootLogin Setting
 
<!-- Add screenshot: sshd_config showing PermitRootLogin no uncommented -->
 
---
 
## Screenshot — SSH Service Running
 
<!-- Add screenshot: systemctl status sshd showing active (running) -->
 
---
 
## Hardening Summary
 
| Control | Implementation | HIPAA Relevance |
|---|---|---|
| SMB signing mandatory | `server/client signing = mandatory` in smb.conf | Transmission security — prevents tampering |
| SMB1 disabled | `server min protocol = SMB2` | Eliminates known critical vulnerabilities |
| NTLM disabled | `ntlm auth = no` | Prevents pass-the-hash credential attacks |
| Host firewall | UFW — deny by default, SMB + SSH only | Access control — limits attack surface |
| Root SSH disabled | `PermitRootLogin no` in sshd_config | Audit control — all admin actions attributed to named accounts |
 
---
 
## Related Documents
 
| Document | Description |
|---|---|
| [01-samba-installation.md](01-samba-installation.md) | Samba installation and base configuration |
| [02-domain-join.md](02-domain-join.md) | Domain join and Kerberos authentication |
| [03-share-configuration.md](03-share-configuration.md) | Share structure and AD group permissions |
