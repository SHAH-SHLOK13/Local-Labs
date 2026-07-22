# Network File System (NFS)

## What NFS Is

NFS is a client-server protocol that lets a machine mount a directory living on another machine over the network and use it as if it were a local folder. The server exports a directory. The client mounts it. From that point on, reading and writing to that mount point is really reading and writing to the server's disk, over the network, mostly invisible to the applications using it.

It runs on port **2049** and was originally built by Sun Microsystems. It's still the default way to share storage between Linux/Unix machines: NAS boxes, shared home directories, clustered web servers pulling from the same asset store, VM storage backends, that kind of thing.

## Why Use It

A few real reasons this comes up:

- **Centralized storage** — instead of copying files to five servers, put them in one place and mount it on all five.
- **Shared home directories** — a user can log into any machine in a lab or cluster and see the same home folder.
- **Simplicity** — no client software to install beyond `nfs-utils`. It's already part of the standard Linux toolkit.
- **Good throughput on a trusted LAN** — NFS assumes the network is reasonably trusted, which is also its biggest weakness (more on that below).

## How It Works

Two roles:

- **Server** — owns the actual files and exposes ("exports") one or more directories to the network.
- **Client** — requests access to an export and mounts it into its own filesystem.

The flow is: client sends a mount request → server checks whether that client (by IP, hostname, or network range) is allowed in the `/etc/exports` file → if allowed, the client gets a handle to that directory and mounts it locally.

NFS authentication is host-based by default, not user-password based. The server trusts the client's IP and the UID/GID numbers the client sends. This matters a lot for the `root_squash` discussion later.

---

## Server Setup

Tested on RHEL/CentOS/Rocky-family systems. Package names differ slightly on Debian/Ubuntu (`nfs-kernel-server` instead of `nfs-utils`), but the exports logic is identical.

### 1. Install the packages

```bash
yum install nfs-utils libnfsidmap
```

`nfs-utils` gives you the server daemons and client tools. `libnfsidmap` handles the name-to-UID mapping used by NFSv4 — it's usually pulled in automatically as a dependency, but no harm calling it out explicitly.

### 2. Open the firewall (don't just kill it)

My original notes here said "disable firewall or iptables if on." That works in a lab, but it's a bad habit to build — you're not securing an NFS share, you're removing the wall around your whole box. The right move is to poke a hole for exactly the services you need:

```bash
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
firewall-cmd --reload
```

If you genuinely need firewalld off for a throwaway lab VM, that's a separate, deliberate decision — don't reach for it as step one.

### 3. Enable and start the required services

```bash
systemctl enable --now nfs-server
systemctl enable --now rpcbind
systemctl enable --now rpc-statd
systemctl enable --now nfs-idmapd
```

(`rpc-statd` handles file locking across reboots, `nfs-idmapd` handles NFSv4 name mapping — both are easy to typo or forget, so it's worth listing them by exact name rather than "and the rest.")

### 4. Create the directory you're going to share

```bash
mkdir /shlokngfs
chmod 777 /shlokngfs
```

`777` is fine for a lab. In anything resembling production, scope this down to the group that actually needs it and rely on `no_root_squash`/`root_squash` plus real ownership instead of throwing the doors open.

### 5. Add the share to `/etc/exports`

For a single client:

```
/shlokngfs  192.168.12.7(rw,sync,no_root_squash)
```

For any client that can reach the server:

```
/shlokngfs  *(rw,sync,no_root_squash)
```

What the options actually mean:

- `rw` — read-write access (use `ro` for read-only shares).
- `sync` — the server confirms a write only after it's actually committed to disk. Slower than `async`, but it means a crash mid-write won't hand the client a false "success." This is the safe default and matches what I had in my notes.
- `no_root_squash` — normally NFS maps the client's root user down to `nobody` on the server (this default behavior is called `root_squash`), so a client's root account can't casually rewrite server files it shouldn't touch. `no_root_squash` turns that protection off, so a root user on the client gets full root privileges on the server's shared files. Useful for a two-box home lab where you control both ends. Genuinely risky on anything you don't fully trust — a compromised client becomes a compromised server.

### 6. Apply the export

This step was missing from my original notes and it's the one that actually makes the `/etc/exports` file take effect without a reboot:

```bash
exportfs -arv
```

`-a` exports everything listed in the file, `-r` re-exports (picks up edits to existing entries), `-v` prints what it's doing so you can confirm the share came up.

### 7. Confirm the share is live

```bash
showmount -e localhost
```

---

## Client Setup

### 1. Install the client tools

```bash
yum install nfs-utils
```

### 2. Enable and start rpcbind

```bash
systemctl enable --now rpcbind
```

### 3. Check the firewall state — don't assume

```bash
systemctl status firewalld
```

Confirm it's either stopped or has the right services allowed, rather than guessing.

### 4. Discover what the server is exporting

```bash
showmount -e 192.168.1.5
```

### 5. Create a mount point

```bash
mkdir /mnt/kramer
```

### 6. Mount the share

```bash
mount 192.168.1.5:/shlokngfs /mnt/kramer
```

### 7. Verify

```bash
df -h
```

You should see the export listed with the server's IP and path as the source.

### 8. Make it survive a reboot (also missing from the original notes)

A manual `mount` disappears the moment the client reboots. To make it permanent, add a line to `/etc/fstab`:

```
192.168.1.5:/shlokngfs   /mnt/kramer   nfs   defaults   0 0
```

### 9. Unmount when you're done

```bash
umount /mnt/kramer
```

---

## Quick Troubleshooting Notes

- **`showmount` hangs or times out** — almost always the firewall on the server side; check that ports for NFS, `rpc-bind`, and `mountd` are actually open, not just that the service is running.
- **"Permission denied" on mount** — check the client IP/range actually matches what's in `/etc/exports`, and that you ran `exportfs -arv` after editing it.
- **Mount succeeds but writes fail** — check the directory permissions on the server side and whether `root_squash` is stripping the privilege you expected.
- **SELinux** — if enabled, you may need `setsebool -P nfs_export_all_rw 1` or the equivalent for your use case; SELinux denials on NFS shares are common and easy to miss if you're only checking firewalld.

---

## Samba (Brief)

Samba is the Linux implementation of the **SMB/CIFS** protocol — the file and printer sharing protocol native to Windows. Where NFS is built around Unix/Linux semantics (UIDs, GIDs, POSIX permissions) and talks comfortably only to other Unix-like systems, Samba's whole purpose is cross-platform: a Samba share on a Linux box shows up as a normal network drive to Windows, macOS, and Linux clients alike, with no extra client software needed.

Rule of thumb: reach for **NFS** when every machine involved is Linux/Unix and you want the simpler, faster, more "native" option. Reach for **Samba** the moment Windows or macOS clients need access, or you need to share printers alongside files.
