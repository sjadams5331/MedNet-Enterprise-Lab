# 01 — Samba Installation
 
## Overview
 
This document covers the base installation of Debian 12 (Bookworm) as a headless server and the initial installation and verification of Samba. This VM serves as the file server for the `mednet.lab` domain.
 
---
 
## VM Configuration
 
Debian 12 was installed in VirtualBox with the following settings:
 
| Setting | Value |
|---|---|
| OS | Debian 12 Bookworm (64-bit) |
| vCPUs | 2 |
| RAM | 4 GB |
| Disk | 50 GB |
| Partitioning | Separate `/home`, `/var`, and `/tmp` partitions |
| Desktop Environment | None (headless server) |
| Packages Selected | SSH server, standard system utilities only |
 
> **Note:** The separate `/var` partition is particularly important for a file server — Samba logs write to `/var/log/samba/` and isolating this partition prevents log growth from impacting the root filesystem or share storage.
 
### Software Selection
 
During install, only the following were selected in tasksel:
 
- ✅ SSH server
- ✅ Standard system utilities
All desktop environments were left unchecked.
 
---
 
## Screenshot — Software Selection
 
<!-- Add screenshot: tasksel with only SSH server and standard system utilities checked -->
 
---
 
## Initial System Configuration
 
### System Updates
 
After first boot, the system was updated as root:
 
```bash
apt update && apt upgrade -y
```
 
### sudo Installation
 
Debian does not install `sudo` by default. It was installed and the `sysadmin` user was added to the sudo group:
 
```bash
apt install sudo -y
usermod -aG sudo sysadmin
```
 
---
 
## Network Configuration
 
The VM was configured with two VirtualBox adapters:
 
| Adapter | Type | Interface | IP |
|---|---|---|---|
| Adapter 1 | NAT | `enp0s3` | DHCP (`10.0.2.15`) |
| Adapter 2 | Host-Only | `enp0s8` | Static (`192.168.56.20`) |
 
The static IP was configured by editing `/etc/network/interfaces`:
 
```
auto enp0s8
iface enp0s8 inet static
    address 192.168.56.20
    netmask 255.255.255.0
```
 
The interface was brought up with:
 
```bash
ifup enp0s8
```
 
---
 
## Screenshot — Network Interface Confirmation
 
<!-- Add screenshot: ip a showing enp0s8 with 192.168.56.20 assigned -->
 
---
 
## Samba Installation
 
Samba was installed from the Debian repositories:
 
```bash
apt install samba -y
```
 
### Version Verification
 
```bash
samba --version
```
 
**Output:**
```
Version 4.17.12-Debian
```
 
---
 
## Screenshot — Samba Install Output
 
<!-- Add screenshot: completed apt install samba -y output -->
 
---
 
## Screenshot — Samba Version
 
<!-- Add screenshot: samba --version output showing 4.17.12-Debian -->
 
---
 
## Service Verification
 
Both Samba daemons were confirmed running after installation:
 
```bash
systemctl status smbd
systemctl status nmbd
```
 
| Service | Role | Status |
|---|---|---|
| `smbd` | SMB daemon — handles file sharing and authentication | Active (running) |
| `nmbd` | NetBIOS name service daemon | Active (running) |
 
Both services are enabled and set to start automatically on boot.
 
---
 
## Screenshot — smbd and nmbd Status
 
<!-- Add screenshot: systemctl status smbd and nmbd both showing active (running) -->
 
---
 
## Related Documents
 
| Document | Description |
|---|---|
| [02-domain-join.md](docs/02-domain-join.md) | Joining the file server to mednet.lab |
| [03-share-configuration.md](03-share-configuration.md) | Hospital share structure and smb.conf |
| [04-security-hardening.md](04-security-hardening.md) | SMB signing, firewall, SSH hardening |
