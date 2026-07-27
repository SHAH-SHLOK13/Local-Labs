# Samba

## What Samba Is

Samba is Linux's implementation of **SMB (Server Message Block)** — the file and printer sharing protocol native to Windows. It lets a Linux box share directories and printers with other operating systems, and it lets a Linux box mount shares coming from a Windows machine too. The relationship goes both ways.

Functionally it looks the same as NFS from a user's point of view: computer A shares a folder, computer B mounts it and sees it as a local drive. The difference is what's on the other end. **NFS assumes both sides are Linux/Unix.** Samba assumes nothing — Windows, macOS, and Linux clients can all talk to the same share with no extra software beyond what's already built into the OS.

## The Protocol Side of Things

A few names get thrown around here, and it's worth being precise about what each one actually is:

- **SMB** — the original protocol, written by IBM in the 1980s.
- **CIFS (Common Internet File System)** — Microsoft's extended version of SMB, released in the late '90s. In casual conversation "SMB" and "CIFS" get used interchangeably, and in practice a CIFS client can talk to an SMB server without issue, because CIFS is SMB with more features bolted on.
- **NMB (NetBIOS Name Server)** — this one gets miscategorized a lot, including in some install guides. NMB isn't a file-sharing protocol at all. It's a name resolution and network-browsing service (the thing that lets `\\centos` resolve to an IP on a local network without DNS). Samba runs `nmb` as a companion service to `smb`, but the actual file transfer happens over SMB/CIFS, not NMB.
- Modern Samba deployments should be running **SMB2 or SMB3**. SMB1 has known, serious vulnerabilities (it's the protocol EternalBlue/WannaCry exploited), and most quick-start guides don't mention disabling it. Worth doing — see the security note near the bottom.

**Ports:** Samba listens on TCP **445** (SMB directly over TCP, what modern clients use) and, for legacy NetBIOS-based name resolution, TCP/UDP **137–139**. If you're only supporting modern clients, 445 is the one that actually matters.

---

## Installation

Take a VM snapshot before you start — same rule as any lab that touches firewall and SELinux config.

### 1. Install the packages

```bash
yum install samba samba-client samba-common
```

### 2. Open the firewall for Samba

```bash
firewall-cmd --permanent --zone=public --add-service=samba
firewall-cmd --reload
```

Some install guides list this step and then, a few lines later, tell you to stop and disable firewalld entirely. Do one or the other, not both — if you've already opened the `samba` service, killing firewalld afterward just throws away the more precise rule you set up. If you're on a lab VM where you genuinely don't care about the firewall, disabling it is a fine shortcut:

```bash
systemctl stop firewalld
systemctl disable firewalld
```

(Note: modern RHEL/CentOS/Rocky releases don't ship a separate `iptables` service by default — firewalld is the front end for netfilter — so `systemctl stop iptables` will just error out with "unit not found" on those systems. It's leftover instruction from older RHEL 6-style guides.)

### 3. Create the share directory and set permissions

```bash
mkdir -p /samba/morepretzels
chmod a+rwx /samba/morepretzels
chown -R nobody:nobody /samba
```

### 4. SELinux context

If SELinux is enforcing (check with `sestatus`), the directory needs the right context or Samba will get permission-denied errors even though the Linux file permissions look fine:

```bash
chcon -t samba_share_t /samba/morepretzels
```

One thing worth adding here: `chcon` only sets the context until the next full relabel — it doesn't persist across a `restorecon -R` or a filesystem relabel. If this share is meant to stick around, set the context properly with `semanage` instead:

```bash
semanage fcontext -a -t samba_share_t "/samba(/.*)?"
restorecon -Rv /samba
```

Disabling SELinux outright is the same story as disabling the firewall — fine for a five-minute lab, not something to build as a habit:

```bash
sestatus
vi /etc/selinux/config
# SELINUX=enforcing  ->  SELINUX=disabled
reboot
```

### 5. Configure `/etc/samba/smb.conf`

Back up the original file before touching it:

```bash
cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

Then replace the contents with:

```ini
[global]
workgroup = WORKGROUP
netbios name = centos
security = user
map to guest = bad user
dns proxy = no
server min protocol = SMB2
client min protocol = SMB2

[Anonymous]
path = /samba/morepretzels
browsable = yes
writable = yes
guest ok = yes
guest only = yes
read only = no
```

`server min protocol = SMB2` and `client min protocol = SMB2` are the addition to the original config — they refuse SMB1 connections outright, which is the recommended baseline for anything not talking to genuinely ancient clients.

### 6. Validate the config

```bash
testparm
```

Fix anything it flags before moving on — a typo in `smb.conf` will silently break a share rather than throw an error at mount time.

### 7. Enable and start the services

```bash
systemctl enable --now smb
systemctl enable --now nmb
```

---

## Connecting as a Client

### From Windows

- Open Start → search bar
- Type `\\192.168.1.95` (your Samba server's IP — check it on the server with `ip addr` or `ifconfig`)

### From Linux

```bash
yum -y install cifs-utils samba-client
mkdir /mnt/sambashare
mount -t cifs //192.168.1.95/Anonymous /mnt/sambashare/
```

Since the `Anonymous` share has `guest ok = yes`, this mounts without prompting for a password.

Before mounting blind, it's worth being able to list what a server is even sharing, the same way `showmount -e` works for NFS:

```bash
smbclient -L //192.168.1.95 -N
```

`-N` skips the password prompt for a guest/anonymous listing.

---

## Securing a Share

The `Anonymous` share above is deliberately open. For anything real, lock a share down to specific users.

### 1. Create a group and a user

```bash
useradd larry
groupadd smbgrp
usermod -a -G smbgrp larry
smbpasswd -a larry
```

`smbpasswd` sets a *Samba* password, separate from `larry`'s regular Linux login password — Samba keeps its own credential store.

### 2. Create the share directory with restricted ownership

```bash
mkdir /samba/securepretzels
chown -R larry:smbgrp /samba/securepretzels
chmod -R 0770 /samba/securepretzels
chcon -t samba_share_t /samba/securepretzels
```

(Apply the same `semanage fcontext` persistence step from earlier if this needs to survive a relabel.)

### 3. Add the share to `smb.conf`

```ini
[Secure]
path = /samba/securepretzels
valid users = @smbgrp
guest ok = no
writable = yes
browsable = yes
```

The original version of this block has `guest ok = no` and `writable = yes` run together on one line (`guest ok = nowritable = yes`) — that's a formatting error, not valid `smb.conf` syntax. They need to be two separate lines like above, or `testparm` will choke on it.

### 4. Restart the services

```bash
systemctl restart smb
systemctl restart nmb
```

### 5. Mount the secure share (missing from the original steps)

Authenticated shares need credentials at mount time — this step wasn't in the original instructions but is the whole point of setting up `larry` and `smbgrp` in the first place:

```bash
mkdir /mnt/securemount
mount -t cifs //192.168.1.95/Secure /mnt/securemount -o username=larry
```

You'll be prompted for the Samba password set with `smbpasswd`. For a mount that needs to persist in `/etc/fstab` without a plaintext password sitting in the file, use a credentials file instead:

```bash
# /root/.smbcredentials (chmod 600)
username=larry
password=YOUR_SAMBA_PASS
```

```
//192.168.1.95/Secure  /mnt/securemount  cifs  credentials=/root/.smbcredentials  0  0
```

---

## NFS vs. Samba, in Short

- **Same machines only, and they're all Linux/Unix** → NFS. Simpler, less overhead, native POSIX permissions.
- **Windows or macOS clients need in, or you're sharing printers too** → Samba. More setup (users, SELinux contexts, `smb.conf`), but it talks to everything.
