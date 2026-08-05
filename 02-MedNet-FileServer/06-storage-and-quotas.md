# Storage & Quotas

## Overview

This document covers how `mednet-fs01` manages the storage underlying the hospital share structure: the LVM-based volume that replaced the original ad-hoc placement of share data, and the per-department disk quotas layered on top of it.

Where `05-backup-and-recovery.md` covers *recovering* data, this document covers *managing the space that data lives on*: ensuring the share structure sits on a volume that can grow as the hospital's storage needs grow, and that no single department can silently consume the entire disk at the expense of every other department sharing it.

---

## Storage Layout: LVM on a Dedicated Data Disk

### Why LVM

A plain partition works until it doesn't: the moment a department outgrows its allocation, a fixed partition means downtime, a full backup/restore cycle, or both. LVM decouples the logical volume from the physical disk boundary, so storage can be extended live, with the filesystem mounted and Samba running, as demonstrated later in this document.

`mednet-fs01` already had a disk provisioned for exactly this purpose (`mednet-fs01-data.vdi`, 20 GB, attached at `SATA Port 1`) from earlier in the build. It had gone unused while other modules were prioritized; the share data was, in the meantime, living directly on the OS root filesystem. This section closes that gap.

> **Note:** an earlier session identified a VirtualBox timer-stall issue tied to the Paravirtualization Interface setting (KVM causing RCU stalls on this AMD host). That was resolved by switching the interface to **Minimal** under VM Settings → System → Acceleration, prior to the work in this document.

### Building the LVM Stack

```bash
sudo apt install lvm2 -y
sudo pvcreate /dev/sdb
```

![Physical volume created on the dedicated data disk](../screenshots/06-storage-and-quotas_01.png)

```bash
sudo vgcreate mednet-data-vg /dev/sdb
```

![Volume group created, ~20 GiB available](../screenshots/06-storage-and-quotas_02.png)

The logical volume was deliberately sized at **15 GB**, not the full 20 GB, leaving roughly 5 GB unallocated in the volume group on purpose, to demonstrate live volume growth later in this document rather than requiring a second disk just to prove the concept:

```bash
sudo lvcreate -L 15G -n samba-data mednet-data-vg
sudo mkfs.ext4 /dev/mednet-data-vg/samba-data
```

![Logical volume created at 15 GiB, formatted ext4](../screenshots/06-storage-and-quotas_03.png)

| Setting | Value |
|---|---|
| Physical volume | `/dev/sdb` (20 GB, `mednet-fs01-data.vdi`) |
| Volume group | `mednet-data-vg` |
| Logical volume | `samba-data`, initially 15 GB |
| Device-mapper path | `/dev/mapper/mednet--data--vg-samba--data` |
| Filesystem | ext4, UUID `15e46774-443b-49d6-902d-346c1453078c` |

---

## Migrating `/srv/samba` onto the LVM Volume

The new volume was mounted temporarily at a scratch path first, rather than directly over the live share, so the data could be copied and verified before anything was cut over:

```bash
sudo mkdir -p /mnt/samba-new
sudo mount /dev/mednet-data-vg/samba-data /mnt/samba-new
```

![New volume formatted and temporarily mounted for migration](../screenshots/06-storage-and-quotas_04.png)

### Preserving the ACL Layer During Copy

The share structure's permission model depends entirely on the POSIX ACLs documented in `03-permissions-and-acls.md`. A plain `cp` would silently drop that layer, so the migration used `rsync` with explicit ACL and extended-attribute preservation:

```bash
sudo rsync -aAX /srv/samba/ /mnt/samba-new/
sudo diff -rq /srv/samba /mnt/samba-new
sudo getfacl /srv/samba/clinical/physicians/intake-form-template.txt
sudo getfacl /mnt/samba-new/clinical/physicians/intake-form-template.txt
```

![rsync migration complete; diff clean aside from lost+found; ACLs verified identical on both copies](../screenshots/06-storage-and-quotas_05.png)

> **Note:** `diff` reported one difference: `lost+found`, a directory `mkfs.ext4` creates automatically on every ext4 filesystem for kernel-level recovery bookkeeping. It's filesystem housekeeping, not share data, and its presence on the new volume (and absence from the comparison baseline) is expected rather than a migration gap.

### Cutover

Samba was stopped, the original data moved aside as a safety net rather than deleted outright, and the new volume mounted at the real path:

```bash
sudo systemctl stop smbd nmbd winbind
sudo umount /mnt/samba-new
sudo mv /srv/samba /srv/samba.old
sudo mkdir -p /srv/samba
sudo mount /dev/mapper/mednet--data--vg-samba--data /srv/samba
```

> **Note:** The friendly LVM path (`/dev/mednet-data-vg/samba-data`) intermittently failed to mount immediately after unmounting, with the kernel reporting the device didn't exist. This was a udev timing quirk rather than a real fault; cycling the volume group (`vgchange -an` then `vgchange -ay`) forced device-mapper nodes to regenerate cleanly. The permanent fstab entry was written using the filesystem's UUID rather than either device-path style, avoiding the issue going forward.

```bash
echo 'UUID=15e46774-443b-49d6-902d-346c1453078c /srv/samba ext4 defaults 0 2' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo umount /srv/samba
sudo mount -a
```

![Fstab corrected to a single UUID-based entry; full unmount/remount-all cycle confirms the LVM volume mounts cleanly at /srv/samba](../screenshots/06-storage-and-quotas_06.png)

Samba was then restarted, not just started, to guarantee a clean process boundary after the mount changes:

```bash
sudo systemctl restart smbd nmbd winbind
sudo systemctl status smbd --no-pager
```

![Samba restarted with a fresh timestamp, confirming a clean process against the new volume](../screenshots/06-storage-and-quotas_07.png)

### Functional Verification

Server-side checks confirm the mount is correct, but the real test is a client actually reading data through Samba. From `WS-CLIN-01`, the `Clinical` share was browsed and `intake-form-template.txt` confirmed reachable and correctly sized:

![File reachable and correctly sized from a domain-joined Windows client, confirming the cutover end-to-end](../screenshots/06-storage-and-quotas_08.png)

Once verified, `/srv/samba.old` was removed:

```bash
sudo rm -rf /srv/samba.old
```

---

## Quota Implementation

### Tooling

```bash
sudo apt install quota -y
sudo quotacheck -V
```

![Quota utilities installed and confirmed](../screenshots/06-storage-and-quotas_09.png)

### Enabling Group Quota Tracking

The `/srv/samba` fstab entry was updated to include `grpquota`, then the filesystem's quota database was built and enforcement switched on:

```bash
sudo sed -i '/\/srv\/samba /s/defaults 0 2/defaults,grpquota 0 2/' /etc/fstab
sudo mount -o remount /srv/samba
sudo quotacheck -cvug /srv/samba
sudo quotaon -v /srv/samba
```

![grpquota mount option applied; quota database built; group quotas turned on](../screenshots/06-storage-and-quotas_10.png)

> **Note:** Both `quotacheck` and `quotaon` warned that external quota files are deprecated on ext4 in favor of the filesystem's native quota feature. An attempt was made to switch to the native method (`tune2fs -O quota`), which failed with `Invalid mount option set: quota`, a mount-option conflict in this environment that wasn't worth extensive detour time to resolve, since the legacy external-file method is still fully functional and widely deployed. The environment was cleanly reverted to the working legacy method rather than left in a half-migrated state:

```bash
sudo mount -a
sudo quotacheck -cvug /srv/samba
sudo quotaon -v /srv/samba
sudo systemctl start smbd nmbd winbind
```

![Reverted cleanly to the legacy quota method after the native ext4 attempt hit a mount-option conflict; group quotas confirmed active again](../screenshots/06-storage-and-quotas_11.png)

### Mapping AD Groups to Quota-Trackable Ownership

Group quotas key off a file's **primary group**, not its ACLs. The share structure deliberately keeps every file owned `root:root` with access granted through named ACL entries (per `02-share-structure.md` and `03-permissions-and-acls.md`), which means, as built, no file's primary group actually matched a department, and a quota on any department group would track nothing.

The fix layers a second mechanism on top without touching the existing one: each department subdirectory's **primary group** was set to its winbind-resolved AD group, with the setgid bit applied so every new file automatically inherits it going forward.

```bash
getent group | grep -iE "clinical|admin-|it-staff|domain users"
```

![Confirmed local GIDs for each AD department group, as resolved by winbind](../screenshots/06-storage-and-quotas_12.png)

```bash
sudo chgrp -R clinical-physicians /srv/samba/clinical/physicians
sudo chmod -R g+s /srv/samba/clinical/physicians
# ...repeated per department...
```

![All eight department directories showing correct group ownership and the setgid bit, with the underlying ACLs still intact](../screenshots/06-storage-and-quotas_13.png)

The result is two independent layers, each doing one job: **ACLs control who can access a share**; **primary group ownership controls whose quota the files count against**. Neither layer depends on or interferes with the other.

> **Note:** The `clinical/shared-clinical` subdirectory is accessible by three separate clinical groups (`clinical-physicians`, `clinical-nursing`, `clinical-pharmacy`). Linux group quotas can only charge usage against one primary group per file, so there's no clean way to quota a genuinely multi-group folder at the department level. It was deliberately left out of the primary-group reassignment rather than forcing an arbitrary single-group answer, a real constraint of Linux's quota model, not an oversight.

### Setting Limits

Limits were differentiated by realistic department storage weight: Clinical carries the heaviest load (records, imaging references), Administrative and IT sit in the middle, and Shared stays intentionally small as a lightweight cross-department space rather than a bulk-storage destination:

| Group | Soft Limit | Hard Limit |
|---|---|---|
| `clinical-physicians` | 3 GB | 3.5 GB |
| `clinical-nursing` | 3 GB | 3.5 GB |
| `clinical-pharmacy` | 2 GB | 2.5 GB |
| `admin-hr` | 1 GB | 1.25 GB |
| `admin-finance` | 1.5 GB | 1.75 GB |
| `admin-reception` | 500 MB | 750 MB |
| `it-staff` | 1 GB | 1.25 GB |
| `domain users` (Shared) | 500 MB | 750 MB |

```bash
sudo setquota -g clinical-physicians 3000000 3500000 0 0 /srv/samba
# ...one setquota call per group...
sudo repquota -g /srv/samba
```

![All eight department quotas confirmed set correctly via repquota](../screenshots/06-storage-and-quotas_14.png)

---

## Quota Enforcement: Tested

To prove the quota system genuinely blocks writes rather than just existing on paper, `admin-reception`'s limit was temporarily lowered and deliberately exceeded.

### A False Start: Root Bypasses Quota

The first attempt wrote data as `root` (via `sudo su`) and the write succeeded in full, despite a hard limit of only 150 KB. This wasn't a broken quota system: Linux processes running as root carry `CAP_SYS_RESOURCE`, a kernel capability that explicitly exempts root from filesystem quota checks. It's expected, documented behavior, but it meant the first test wasn't actually testing enforcement:

![Write completed in full as root, despite a 150 KB hard limit, showing the quota bypass in action](../screenshots/06-storage-and-quotas_15.png)

The test was corrected to run as a genuine non-root user (`sudo -u sysadmin -g admin-reception ...`), which is what a real department staff member's write would look like.

### Enforcement Confirmed

The corrected first write actually succeeded too, but the reason turned out to be informative rather than a failure of the setup. The legacy external-quota-file method enforces at **writeback** time, not at the moment `write()` is called, so a buffered write can complete before the kernel's quota check catches up. This is precisely the behavior the earlier deprecation warning was pointing at: the native ext4 quota feature enforces at block-allocation time, which is tighter. Forcing a sync surfaced the over-limit state immediately after:

```bash
sync
sudo repquota -g /srv/samba | grep admin-reception
sudo -u sysadmin -g admin-reception dd if=/dev/urandom of=/srv/samba/administrative/reception/quota-test2.bin bs=1K count=10
```

![Group confirmed over its hard limit with an active grace period; the next write is rejected outright with "Disk quota exceeded"](../screenshots/06-storage-and-quotas_16.png)

The test files were removed and `admin-reception`'s limit restored to its real 500 MB/750 MB values afterward.

---

## Growth Path: Live Volume Extension

This is the payoff for the 5 GB deliberately left unallocated when the logical volume was created: LVM allows growing storage without downtime, without unmounting, and without interrupting the running Samba service.

> **Note:** a first attempt at `lvextend -L +5G` failed with *"Insufficient free space: 1280 extents needed, but only 1279 available"*. The volume group's true free space was fractionally under 5 GB due to LVM's own metadata reservation. Rather than calculate the exact remaining size, `-l +100%FREE` was used instead, which claims whatever is actually available regardless of the precise figure.

```bash
sudo lvextend -l +100%FREE /dev/mapper/mednet--data--vg-samba--data
sudo resize2fs /dev/mapper/mednet--data--vg-samba--data
```

![Logical volume grown from 15 GiB to ~20 GiB and the filesystem resized online; /srv/samba now shows ~20 GB total; smbd's uptime is unbroken across the entire operation](../screenshots/06-storage-and-quotas_17.png)

`resize2fs` explicitly confirmed this was an **online** resize (filesystem mounted, Samba actively running), and `smbd`'s process uptime carried through the entire operation without a restart. This is the concrete argument for LVM over a fixed partition: a department outgrowing its share doesn't require scheduled downtime to fix.

---

## Monitoring Pointer

Quota usage and volume capacity are worth alerting on before they become a hard block a user hits unexpectedly. That alerting configuration lives in [`04-MedNet-NetworkMonitoring`](../../04-MedNet-NetworkMonitoring/README.md) rather than duplicated here, keeping monitoring logic in the module that owns it.

---

## Limitations & Production Considerations

- **`clinical/shared-clinical` has no clean quota owner.** As noted above, Linux group quotas track one primary group per file, and this folder is genuinely multi-group by design. A production system might address this with project quotas (XFS) or application-level storage tracking instead of relying on the Linux group-quota primitive.
- **Legacy quota method, not native ext4 quota.** The native feature was attempted and reverted after a mount-option conflict. The legacy method is fully functional and widely used, but enforces at writeback rather than at write time, a real, if narrow, timing gap that the native method closes.
- **Block quotas only, not inode quotas.** Limits were set on space consumed (`0 0` for the inode fields), not file count. For this share's usage pattern (documents, forms, records), space is the meaningful constraint; a share expecting very large numbers of tiny files might want inode limits too.
- **No automated capacity alerting yet.** `repquota` and `df` show current state on demand, but nothing currently pages an administrator as a group approaches its limit. See the Monitoring Pointer above.

---

## Related Documents

| Document | Description |
|---|---|
| [01-ad-integration.md](docs/01-ad-integration.md) | Domain join, Kerberos authentication, and AD identity resolution |
| [02-share-structure.md](docs/02-share-structure.md) | Share layout, on-disk structure, and `smb.conf` share definitions |
| [03-permissions-and-acls.md](docs/03-permissions-and-acls.md) | POSIX ACLs, group-based read/write control, and the cross-OS access demonstration |
| [04-security-hardening.md](docs/04-security-hardening.md) | SMB signing, protocol hardening, firewall, and SSH hardening |
| [05-backup-and-recovery.md](docs/05-backup-and-recovery.md) | Backup method, retention, and a tested restore |
| [MedNet-ActiveDirectory/01-domain-design.md](../../01-MedNet-ActiveDirectory/docs/01-domain-design.md) | AD OU structure, security groups, and user accounts |

---

*Part of the [MedNet Enterprise Lab](../README.md), an Enterprise Healthcare IT Infrastructure & Security Operations home lab.*
