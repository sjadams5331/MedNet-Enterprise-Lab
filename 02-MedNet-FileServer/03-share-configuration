# 03 — Share Configuration
 
## Overview
 
This document covers the hospital department share structure deployed on `mednet-fs01`. Each share maps to a department in the `mednet.lab` AD organizational structure and is restricted to the corresponding AD security group, enforcing role-based access control consistent with HIPAA minimum necessary access principles.
 
---
 
## Design Rationale
 
In a real hospital environment, file shares are scoped by department to prevent unauthorized access to sensitive data. For example:
 
- Clinical staff should not access HR or Finance files
- Physicians may need access to patient-related clinical data that nursing staff also access, but through separate controlled paths
- IT staff require access to infrastructure documentation not appropriate for clinical users
This share structure mirrors that model using AD security groups as the access control mechanism. No local Samba users are created — all authentication flows through `mednet.lab` via Kerberos.
 
---
 
## AD Security Groups
 
The following security groups were created in `OU=Security Groups,DC=mednet,DC=lab` on the AD server before share configuration:
 
| Group Name | Department | Members |
|---|---|---|
| `Clinical-Physicians` | Clinical / Physicians | `s.mitchell`, `j.ortega` |
| `Clinical-Nursing` | Clinical / Nursing | `l.nguyen` |
| `Clinical-Pharmacy` | Clinical / Pharmacy | `m.evans` |
| `Admin-HR` | Administrative / HR | `k.booth` |
| `Admin-Finance` | Administrative / Finance | `t.reyes` |
| `Admin-Reception` | Administrative / Reception | `d.cole` |
| `IT-Staff` | IT Department | `a.turner` |
 
Groups were created on the Windows Server using PowerShell:
 
```powershell
New-ADGroup -Name "Clinical-Physicians" -GroupScope Global -GroupCategory Security -Path "OU=Security Groups,DC=mednet,DC=lab"
# (repeated for each group)
```
 
Users were added to their respective groups:
 
```powershell
Add-ADGroupMember -Identity "Clinical-Physicians" -Members "s.mitchell","j.ortega"
# (repeated for each group)
```
 
---
 
## Screenshot — AD Security Groups Created
 
<!-- Add screenshot: PowerShell output showing all New-ADGroup commands completed successfully -->
 
---
 
## Screenshot — Group Membership Verification
 
<!-- Add screenshot: Get-ADGroupMember -Identity "Clinical-Physicians" showing Dr. Sarah Mitchell and Dr. James Ortega -->
 
---
 
## Share Directory Structure
 
Share directories were created under `/srv/shares/`:
 
```bash
mkdir -p /srv/shares/{physicians,nursing,pharmacy,hr,finance,reception,it,shared}
chmod -R 770 /srv/shares/
```
 
```
/srv/shares/
├── physicians/
├── nursing/
├── pharmacy/
├── hr/
├── finance/
├── reception/
├── it/
└── shared/
```
 
> **Note:** `/srv/shares/` follows the Linux filesystem standard — `/srv` is the conventional location for data served by the system. This is preferable to paths like `/home/shares/` for a production-style deployment.
 
---
 
## Screenshot — Share Directories
 
<!-- Add screenshot: ls -la /srv/shares/ showing all 8 directories -->
 
---
 
## smb.conf Share Definitions
 
The following share definitions were added to `/etc/samba/smb.conf` below the `[global]` section:
 
```ini
[physicians]
   path = /srv/shares/physicians
   valid users = @"MEDNET\Clinical-Physicians"
   read only = no
   browseable = yes
   create mask = 0660
   directory mask = 0770
 
[nursing]
   path = /srv/shares/nursing
   valid users = @"MEDNET\Clinical-Nursing"
   read only = no
   browseable = yes
   create mask = 0660
   directory mask = 0770
 
[pharmacy]
   path = /srv/shares/pharmacy
   valid users = @"MEDNET\Clinical-Pharmacy"
   read only = no
   browseable = yes
   create mask = 0660
   directory mask = 0770
 
[hr]
   path = /srv/shares/hr
   valid users = @"MEDNET\Admin-HR"
   read only = no
   browseable = yes
   create mask = 0660
   directory mask = 0770
 
[finance]
   path = /srv/shares/finance
   valid users = @"MEDNET\Admin-Finance"
   read only = no
   browseable = yes
   create mask = 0660
   directory mask = 0770
 
[reception]
   path = /srv/shares/reception
   valid users = @"MEDNET\Admin-Reception"
   read only = no
   browseable = yes
   create mask = 0660
   directory mask = 0770
 
[it]
   path = /srv/shares/it
   valid users = @"MEDNET\IT-Staff"
   read only = no
   browseable = yes
   create mask = 0660
   directory mask = 0770
 
[shared]
   path = /srv/shares/shared
   valid users = "@MEDNET\Domain Users"
   read only = no
   browseable = yes
   create mask = 0660
   directory mask = 0770
```
 
### Permission Explanation
 
| Setting | Value | Meaning |
|---|---|---|
| `valid users` | `@"MEDNET\GroupName"` | Only members of this AD group can connect |
| `read only` | `no` | Users can read and write |
| `create mask` | `0660` | New files are readable/writable by owner and group only |
| `directory mask` | `0770` | New directories are accessible by owner and group only |
 
The `@` prefix tells Samba to treat the value as a group rather than a user. The `MEDNET\` prefix specifies the domain.
 
---
 
## Configuration Validation
 
The configuration was validated with:
 
```bash
testparm
```
 
**Expected output:**
```
Loaded services file OK.
Server role: ROLE_DOMAIN_MEMBER
```
 
---
 
## Screenshot — testparm Output
 
<!-- Add screenshot: testparm showing "Loaded services file OK" and "Server role: ROLE_DOMAIN_MEMBER" -->
 
---
 
## Service Restart
 
After configuration, Samba was restarted:
 
```bash
systemctl restart smbd nmbd
systemctl status smbd
```
 
---
 
## Screenshot — smbd Running with Shares Loaded
 
<!-- Add screenshot: systemctl status smbd showing active (running) with share definitions visible in dump -->
 
---
 
## Related Documents
 
| Document | Description |
|---|---|
| [02-domain-join.md](02-domain-join.md) | Domain join and Kerberos authentication |
| [04-security-hardening.md](04-security-hardening.md) | SMB signing and access hardening |
| [MedNet-ActiveDirectory/01-domain-design.md](../MedNet-ActiveDirectory/01-domain-design.md) | AD OU structure and user accounts |
