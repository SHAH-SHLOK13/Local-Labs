# Implementing Advanced Storage Features: Stratis
 
## What is Stratis?
 
RedHat introduces **Stratis** as a local storage management solution. Unlike traditional LVM, Stratis combines two separate processes into one unified layer:
 
1. Creating and managing logical volumes (LVM)
2. Creating and managing the filesystem on top of them
### LVM vs Stratis
 
| | LVM | Stratis |
|---|---|---|
| Filesystem full | Must be extended **manually** | Extends **automatically** if the pool has available space |
| Management | Two separate layers (LV + filesystem) | Single unified layer |
 
**In short:** with LVM, when a filesystem fills up you have to step in and grow it yourself. With Stratis, as long as the underlying pool has free space, the filesystem grows on its own.
 
### How it fits together
 
```
Disk 1 (10G) ─┐
Disk 2 (10G) ─┼──▶ Pool (30G) ──▶ Filesystem (10G, thin-provisioned)
Disk 3 (10G) ─┘
```
 
If the filesystem gets full, it will **automatically extend** up to the pool size — no manual `lvextend` / `resize2fs` dance required.
 
---
 
## 1. Installation & Setup
 
```bash
# Install Stratis packages
dnf install stratis-cli stratisd -y
 
# Enable and start the Stratis daemon
systemctl enable --now stratisd
```
 
---
 
## 2. Preparing Disks
 
Added 2 x 5G new disks from the virtualization software (in my case, Oracle VirtualBox → storage settings).
 
```bash
lsblk   # verify the new disks are visible
```
 
---
 
## 3. Creating & Managing a Pool
 
```bash
# Create a pool
stratis pool create pool1 /dev/sdc
 
# Verify the pool
stratis pool list
 
# Add more data/disks to an existing pool (extend it)
stratis pool add-data pool1 /dev/sde
 
# Verify the pool was extended
stratis pool list
```
 
---
 
## 4. Creating & Mounting a Filesystem
 
```bash
# Create a new filesystem inside the pool
stratis filesystem create pool1 fs1
 
# Verify
stratis filesystem list
 
# Make a mount point and mount it
mkdir /bigdata
mount /dev/stratis/pool1/fs1 /bigdata
 
# Verify
lsblk
df -h
```
 
### ⚠️ Important gotcha: thin provisioning
 
`df -h` will show the Stratis filesystem size **as if** it's fully available — but it isn't actually allocated that way. It's just what the filesystem reports.
 
To see the **real, actual size** consumed on the pool, use:
 
```bash
stratis filesystem list
```
 
This shows the actual size used, since the default behavior of Stratis is **thin provisioning** — space is only truly consumed as data is written, not reserved upfront.
 
---
 
## 5. Snapshots
 
```bash
# Create a snapshot of an existing filesystem
stratis filesystem snapshot pool1 fs1 fs1-snap
 
# Verify — both fs1 and fs1-snap will show up, each with their own UUID
stratis filesystem list
```
 
---
 
## 6. Mounting at Boot (fstab entry)
 
To make sure the Stratis filesystem mounts automatically at boot, add an entry like this to `/etc/fstab`:
 
```
UUID=<your-uuid-here>   /bigdata   /stratis   defaults,x-systemd.requires=stratisd.service   0   0
```
 
The `x-systemd.requires=stratisd.service` option is key — it ensures the `stratisd` daemon is up and running before systemd attempts the mount.
 
---
